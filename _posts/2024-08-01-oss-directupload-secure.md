---
layout: post
title: 更安全的OSS客户端直传方式实践
date: 2024-08-01 09:59
description: OSS直传, 如果更安全, 避免客户端乱来?
categories: aliyun
comments: true
tags:
  - aliyun,oss
---

在典型的服务端和客户端架构下，常见的文件上传方式是服务端代理上传：客户端将文件上传到业务服务器，然后业务服务器将文件上传到OSS。在这个过程中，一份数据需要在网络上传输两次，会造成网络资源的浪费，增加服务端的资源开销。为了解决这一问题，您可以在客户端直连OSS来完成文件上传，无需经过业务服务器中转。

![为什么需要客户端直传](/img/ossupload/clientupload.png )

官方文档其实都写了, 只是不仔细看的话,很难搞出来最安全的组合 :

* [官方文章: 在客户端直接上传文件到OSS](https://help.aliyun.com/zh/oss/use-cases/uploading-objects-to-oss-directly-from-clients/){:target="_blank"}
* [官方文章: 服务端签名直传](https://help.aliyun.com/zh/oss/use-cases/obtain-signature-information-from-the-server-and-upload-data-to-oss){:target="_blank"}

上传文件到OSS需要使用RAM用户的访问密钥（AccessKey）来完成签名认证，但是在客户端中使用长期有效的访问密钥，可能会导致访问密钥泄露，进而引起安全问题。

目前一共有以下几种方式可以让用户直传到OSS:

* 客户端直接使用长期AK,SK上传
* 服务端使用长期AK生成STS临时访问凭证
* 服务端使用长期AK生成PostObject所需的签名和Post Policy
* 服务端使用长期AK生成PutObject所需的签名URL
* 服务端使用长期AK生成STS临时访问凭证, 然后使用临时凭证生成PostObject所需的签名和Post Policy

客户端持有长期AK/SK这种方式肯定不能用; 签名URL这种比较简单, 本文不考虑. 以下表格是其中三种方式的对比.

| 协议        | Post Policy                     | STS Policy                                                                                         | STS + Post Policy                 |
|-----------|---------------------------------|----------------------------------------------------------------------------------------------------|-----------------------------------|
| 安全性       | 相对安全, 会暴露长期AK. 可以加较多限制          | 相对安全, 会暴露临时AK, Secret.<br/> 因为要多次使用, 必须放开某个目录权限.<br/> 需客户端自己控制避免覆盖文件,<br/> 无法控制文件类型, callback等字段限制 | 安全, 仅暴露临时AK, Token. 可以加较多限制       |
| 请求频率      | 可以每次上传申请                        | 必须缓存Policy(一般建议业务调用方使用服务端缓存, 页面JS可能生命周期较短), 频繁调用STS服务会引起限流; <br/>客户端: 过期后重新获取(一般为30分钟, 需和服务提供方确认)  | 可以每次上传申请, 服务器端: 缓存Sts Token并做过期检查 |
| 断点续传/分片上传 | 不支持                             | 支持                                                                                                 | 不支持                               |
| 使用方式      | 通过构建HTML表单上传的方式上传文件(JS, Java均可) | 可以使用 OSS SDK上传(JS, Java都有SDK)                                                                      | 通过构建HTML表单上传的方式上传文件(JS, Java均可)   |


我们可以看到, 如果需要使用"断点续传/分片上传", 则只能使用STS Policy方式, 但是无法做过多的细节限制.
而简单的Post Policy方式, 会直接暴露长期AK给客户端. 而使用 STS + Post Policy的方式, 暴露给客户端的是临时AK和比较严格的Post Policy, 一举二得, 就是生成Policy的代码有点多.

那我们希望对客户端做哪些控制哪
* 不希望客户端覆盖已有文件
* 控制上传的文件类型(例如图像)
* 只能上传到指定的目录下
* 不能设置文件的权限为public
* 必须设置callback, 告诉服务端上传了文件. 否则服务端不知道有人上传了文件
* 申请Policy或callback时记录用户信息, 以备后续查询.
* 限制上传文件大小

我们先看看如何生成这个Policy, 流程如下:

![生成流程](/img/ossupload/stspost-flow.png )

### 首先是生成STS Token临时凭证

```java

/**
 * STS Service. for test.
 *
 * @author Felix Zhang 2024-07-22 14:34
 * @version 1.0.0
 */
@Service
@Slf4j
public class StsServiceImpl {

    //注: 以下配置使用Spring的属性自动设置

    @Value("${sts.endpoint}")
    private String stsEndpoint;
    @Value("${sts.regionId}")
    private String stsRegionId;

    @Value("${sts.accessKeyId}")
    private String stsAccessKeyId;
    @Value("${sts.accessKeySecret}")
    private String stsAccessKeySecret;

    @Value("${sts.roleSessionName}")
    private String roleSessionName;
    @Value("${sts.role-arn}")
    private String roleArn;

    @Value("${sts.stspostrampolicy}")
    private String stspostrampolicy;
    @Value("${sts.stsrampolicy}")
    private String stsrampolicy;

    @Value("${sts.durationSeconds}")
    private Long durationSeconds;

    @Value("${oss.tempPath}")
    private String tempPath;

    @Value("${oss.publicTempPath}")
    private String publicTempPath;
    
    @Value("${oss.bucket.tmp}")
    private String tmpBucket;

    @Value("${oss.endpoint}")
    private String endpoint;

    Map<String, StsCredentialInfo> stsTokenCache = new HashMap<>();

    //获取可用的StsToken
    public StsCredentialInfo fetchUsableStsToken(boolean forPostPolicy, String source, 
                                                 String fileLevel, boolean forcenewpolicy) 
            throws Exception {
        String stsTokenCacheKey = source + "_" + fileLevel;

        boolean ok = false;
        StsCredentialInfo theInfo = null;
        if (!forcenewpolicy) {
            theInfo = stsTokenCache.get(stsTokenCacheKey);
            if (theInfo != null) {
                //如果仅仅是使用普通的StsPolicy, 是否还保留30分钟冗余哪?
                //检查是否过期保证期内 30分钟: 留给Post Signature 30分钟, 所以只有30分钟了
                if ((theInfo.getExpire() * 1000 + 1800 * 1000) < System.currentTimeMillis()) {
                    ok = true;
                }
            }
        }
        if (!ok) {
            theInfo = generateStsToken(forPostPolicy, source, fileLevel);
        }

        return theInfo;
    }

    //按source+fileLevel: 缓存到一个Map
    private StsCredentialInfo generateStsToken(boolean forPostPolicy, String source, 
                                               String fileLevel) 
            throws Exception {
        // Init ACS Client
        DefaultProfile.addEndpoint("", "", "Sts", stsEndpoint);

        IClientProfile profile = DefaultProfile.getProfile(stsRegionId, stsAccessKeyId, stsAccessKeySecret);
        DefaultAcsClient client = new DefaultAcsClient(profile);

        // Optional
        final AssumeRoleRequest request = new AssumeRoleRequest();
        request.setMethod(MethodType.POST);
        request.setRoleArn(roleArn);
        request.setRoleSessionName(roleSessionName);
        request.setDurationSeconds(durationSeconds);

        //设置虚拟角色的Policy, 缩小权限
        String mypolicy = dynamicCreateRamPolicy(forPostPolicy, source);
        request.setPolicy(mypolicy);

        log.info("ram policy: \n" + mypolicy);

        final AssumeRoleResponse response = client.getAcsResponse(request);

        // set result
        AssumeRoleResponse.Credentials credentials = response.getCredentials();
        StsCredentialInfo stsCredentialInfo = new StsCredentialInfo();
        stsCredentialInfo.setAccessKeyId(credentials.getAccessKeyId());
        stsCredentialInfo.setAccessKeySecret(credentials.getAccessKeySecret());
        stsCredentialInfo.setSecurityToken(credentials.getSecurityToken());
        stsCredentialInfo.setExpiration(credentials.getExpiration());

        Date date = DateUtil.parseIso8601Date(credentials.getExpiration());
        stsCredentialInfo.setExpire( date.getTime() / 1000);

        //Post Signature不需要, 它会自己设置
        //String callbackStr = generateCallback();
        //stsCredentialInfo.setCallback(callbackStr);

        // 指定ossKey, 加上业务名
        if (fileLevel.equals("privateFile")) {
            String ossKey = tempPath + "/" + source + "/";
            stsCredentialInfo.setBucketName(tmpBucket);
            stsCredentialInfo.setUploadPath(ossKey);
            //ossTmpTokenDTO.setFileId(IdWorker.getId()); //干嘛用?!
        }
        //todo 注意区分图像和文件
        else if (fileLevel.startsWith("public")) {
            String ossKey = publicTempPath + "/" + source + "/";
            stsCredentialInfo.setBucketName(tmpBucket);
            stsCredentialInfo.setUploadPath(ossKey);
        }
        stsCredentialInfo.setEndPoint(endpoint);

        String stsTokenCacheKey = source + "_" + fileLevel;
        stsTokenCache.put(stsTokenCacheKey, stsCredentialInfo);

        return stsCredentialInfo;
    }

    private String dynamicCreateRamPolicy(boolean forPostPolicy, String source) {

        String theRamPolicy = stsrampolicy;
        if(forPostPolicy){
            theRamPolicy = stspostrampolicy;
        }

        //动态替换
        return theRamPolicy.replace("{source}", source);
    }

    
}

```
其中使用的临时角色权限设定如下(stspostrampolicy): 
```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "oss:PutObject"
      ],
      "Resource": [
              "acs:oss:*:*:xxx-test-tmp/resource-temp/{source}/*",
              "acs:oss:*:*:xxx-test-pub/openfile-temp/{source}/*",
              "acs:oss:*:*:xxx-test-pub/openimage-temp/{source}/*"
      ]
    },
    {
      "Effect": "Deny",
      "Action": [
        "oss:PutObject"
      ],
      "Resource": [
            "acs:oss:*:*:xxx-test-tmp/resource-temp/{source}/*",
            "acs:oss:*:*:xxx-test-pub/openfile-temp/{source}/*",
            "acs:oss:*:*:xxx-test-pub/openimage-temp/{source}/*"
      ],
      "Condition": {
        "StringEquals": {
          "oss:x-oss-object-acl": [
            "public-read",
            "public-read-write"
          ]
        }
      }
    } 
    
  ]
}
```

可以看到只有PubObject的权限, 而且只能传到临时目录下. (后续通过其他调用移动到正式目录下); 而且只能是private权限的文件.



### 然后根据临时凭证生成Post Policy
```java
    
    @Value("${oss.endpoint}")
    private String endpoint;
    @Value("${oss.accessKeyId}")
    private String accessKeyId;
    @Value("${oss.accessKeySecret}")
    private String accessKeySecret;
    
    @Value("${oss.bucket.tmp}")
    private String tmpBucket;
    @Value("${oss.tempPath}")
    private String tempPath;
    @Value("${oss.publicTempPath}")
    private String publicTempPath;
    
    @Value("${oss.defaultFolder.tmp}")
    private String defaultTempFolder;
    @Value("${oss.callbackUrl}")
    private String callbackUrl;
    
    @Value("${oss.defaultFolder.resourceTemp}")
    private String defaultTemporaryFolder;
    
    
    private OSSClient ossClient;


    public OSSClient newClient(String theAccessKeyId, String theAccessKeySecret) {
        ClientConfiguration conf = new ClientConfiguration();
        return new OSSClient(endpoint, new DefaultCredentialProvider(theAccessKeyId, theAccessKeySecret), conf);
    }

    public Map<String, String> generatePostPolicy(String fileLevel, String md5, 
                                                  String source, String requestId) throws Exception {
        return generatePostPolicy(ossClient, fileLevel, md5, source, requestId);
    }

    public Map<String, String> generatePostPolicy(OSSClient theClient, String fileLevel, String md5, 
                                                  String source, String requestId) throws Exception {

        String bucket = tmpBucket;
        if (StringUtils.isNotBlank(fileLevel) && (fileLevel.equals("publicFile") || fileLevel.equals("publicImage"))) {
            //Todo 注意: 此处使用xxx-pub这个bucket
            bucket = tmpBucket;
        }

        String host = "https://" + bucket + "." + endpoint;

        // 用户上传文件时指定的前缀, 统一上传到临时path中
        // 限定业务目录, private/public** 实际情况会不一样, 注意修改
        String dir = tempPath + "/" + source + "/";

        //todo 注意区分图像和文件
        if (fileLevel.startsWith("public")) {
            dir = publicTempPath + "/" + source + "/";
        }

        Map<String, String> respMap = null; //返回结果

        try {
            long expireTime = 1800;
            long expireEndTime = System.currentTimeMillis() + expireTime * 1000;
            Date expiration = new Date(expireEndTime);

            // PostObject请求最大可支持的文件大小为5 GB，即CONTENT_LENGTH_RANGE为5*1024*1024*1024。
            PolicyConditions policyConds = new PolicyConditions();
            policyConds.addConditionItem(PolicyConditions.COND_CONTENT_LENGTH_RANGE, 0, 5 * 1024 * 1024 * 1024L);
            policyConds.addConditionItem(MatchMode.StartWith, PolicyConditions.COND_KEY, dir);

            //限定bucket
            policyConds.addConditionItem("bucket", bucket);

            Map<String, Object> callback = getCallback(md5, source, requestId);
            String callbackData = BinaryUtil.toBase64String(JSONUtil.parse(callback).toString().getBytes(StandardCharsets.UTF_8));

            //增加callback限制条件   
            policyConds.addConditionItem("callback", callbackData);

            //强制要求 x-oss-forbid-overwrite   
            policyConds.addConditionItem("x-oss-forbid-overwrite", "true");

            //限制图像类型 注意补全图片类型   
            if (fileLevel.equals("publicImage")) {
                policyConds.addConditionItem(MatchMode.In, COND_CONTENT_TYPE, images);
            }

            log.info("policy cond: " + policyConds.jsonize());

            //生成policy输出字符串
            String postPolicy = theClient.generatePostPolicy(expiration, policyConds);
            byte[] binaryData = postPolicy.getBytes(StandardCharsets.UTF_8);
            String encodedPolicy = BinaryUtil.toBase64String(binaryData);
            String postSignature = theClient.calculatePostSignature(postPolicy);


            respMap = new LinkedHashMap<String, String>();
            respMap.put("accessid", theClient.getCredentialsProvider().getCredentials().getAccessKeyId());
            respMap.put("policy", encodedPolicy);
            respMap.put("signature", postSignature);
            respMap.put("dir", dir);
            respMap.put("host", host);
            respMap.put("policyexpire", String.valueOf(expireEndTime / 1000));
            respMap.put("callback", callbackData);
            respMap.put("bucket", bucket);
        }
        catch (Exception e) {
            log.error("获取oss签名失败：" + e);
            throw new Exception(e);
        }
        return respMap;
    }

    //callback限定
    private Map<String, Object> getCallback(String md5, String source, String requestId) {
        String sourceId =  source + "_" + requestId;
        Map<String, Object> callback = new HashMap<>();
        callback.put("callbackUrl", callbackUrl);
        callback.put("callbackBody", ("{\"mimeType\":${mimeType},\"contentMd5\":${contentMd5},\"size\":${size},\"bucket\":${bucket},\"object\":${object}," +
                "\"md5\":\"${md5}\",\"userparamuserid\":${x:userparamuserid},\"sourceId\":\"${sourceId}\"}")
                .replace("${md5}", Objects.nonNull(md5) ? md5 : "").replace("${sourceId}", sourceId));
        callback.put("callbackBodyType", "application/json");
        return callback;
    }

```

可以看到我们在Post Policy里增加了文件大小, 目录, bucket, callback, 强制覆盖参数, 图像类型等强制限制, 如果客户端不遵循规则, 则会被拒绝上传, 保证了OSS服务的安全有序.


下面是组合输出给客户端的组合代码
```java
    /**
     * 使用角色扮演临时AK生成 Post Policy. (既不暴露长期AK, 也能做强制性的细粒度限制; 需要考虑二个过期时间的协调问题)
     */
    @RequestMapping("/stspostpolicy")
    public ApiResult stsPostPolicy(@RequestParam(defaultValue = "labtest") String source, String md5,
                                                      @RequestParam(defaultValue = "privateFile") String fileLevel,
                                                      @RequestParam(required = false) boolean forcenewpolicy) {

        String uuid = UUID.randomUUID().toString();

        StsCredentialInfo stsCredentialInfo;
        OSSClient ossClient = null;
        try {
            //
            stsCredentialInfo = stsService.fetchUsableStsToken(true, source, fileLevel, forcenewpolicy);

            //临时client
            ossClient = ossClientService.newClient(stsCredentialInfo.getAccessKeyId(), stsCredentialInfo.getAccessKeySecret());

            //根据StsPolicy生成PostSignature
            Map<String, String> map = ossClientService.generatePostPolicy(ossClient, fileLevel, md5, source, uuid);

            //增补信息
            map.put("securityToken", stsCredentialInfo.getSecurityToken());
            map.put("stsTokenExpiration", stsCredentialInfo.getExpiration()); //token的过期时间, 用作排查
            map.put("stsTokenExpire", String.valueOf(stsCredentialInfo.getExpire()));


            return ApiResult.succeed(UrcResultCode.RESULT_CODE__SUCCESS, "请求临时权限成功", map).setRequestId(uuid);
        }
        catch (Exception e) {
            log.error("请求临时权限失败", e);
            return ApiResult.succeed(UrcResultCode.RESULT_CODE__INNER_ERROR, "请求临时权限失败," + e, null).setRequestId(uuid);
        }
        finally {
            if (ossClient != null) {
                ossClient.shutdown();
            }
        }
    }
```


然后客户端(JS, Java都可以是客户端)就可以根据Policy直接上传文件到OSS了, 但不能违反STS临时凭证和Post Policy的限制.

### JS客户端版本
```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>上传文件到OSS</title>
</head>
<body>
<div class="container">
  <form>
    <div class="mb-3">
      <label for="file" class="form-label">选择文件</label>
      <input type="file" class="form-control" id="file" name="file" required/>
    </div>
    <button type="submit" class="btn btn-primary">上传</button>
  </form>
</div>
<script type="text/javascript">
    const form = document.querySelector("form");
    const fileInput = document.querySelector("#file");
    form.addEventListener("submit", (event) => {
        event.preventDefault();
        const file = fileInput.files[0];
        const filename = fileInput.files[0].name;

        fetch("/lab/stspostpolicy?source=labtest2&fileLevel=publicImage", {method: "POST"})
            .then((response) => {
                if (!response.ok) {
                    throw new Error("获取签名失败");
                }
                return response.json();
            })
            .then((res) => {
                const data = res.data;
                const formData = new FormData();
                formData.append("OSSAccessKeyId", data.accessid);
                formData.append("signature", data.signature);

                //sts token
                formData.append("x-oss-security-token", data.securityToken);

                //post policy
                formData.append("policy", data.policy);
                formData.append("success_action_status", "200");

                formData.append("name", filename);
                formData.append("key", data.dir + filename);

                //测试: 如果没有callback 或错误callback,  测试成功, 符合预期
                formData.append("callback", data.callback)

                //callback var  有三种设置方式 https://help.aliyun.com/zh/oss/developer-reference/callback
                //变量必须为小写: 测试可行
                formData.append("x:userparamuserid", "123456");

                //测试:  x-oss-forbid-overwrite  测试成功, 符合预期
                formData.append("x-oss-forbid-overwrite", "true")

                //测试 content-type: 限制成功, 符合预期

                formData.append("file", file);

                return fetch(data.host, {method: "POST", body: formData});
            })
            .then((response) => {
                if (response.ok) {
                    console.log("上传成功");
                    alert("文件已上传");
                } else {
                    console.log("上传失败", response);
                    alert("上传失败，请稍后再试");
                }
            })
            .catch((error) => {
                console.error("发生错误:", error);
            });
    });
</script>
</body>
</html>

```

### Java客户端版本
```java

@RestController
@RequestMapping("/lab/clientdemo")
@Slf4j
public class StsPostPolicyUploadDemo {

    public static final String host = "http://127.0.0.1:9006"; //本地

    public static final String upload_policy_path = "/lab/stspostpolicy";

    private final CloseableHttpClient httpClient = HttpClients.createDefault();

    //临时文件, 用来测试上传
    String resourceFilePath = "/tempfile/sample01.mp4";

    @RequestMapping("/stspost")
    public String testStsPostPolicyUpload() throws Exception {
        //获取上传policy接口
        String policyResult = fetchUploadPolicy();
        JSONObject jsonObject = JSONObject.parseObject(policyResult);

        boolean success = jsonObject.getBoolean("success");
        if (!success) {
            return "failed";
        }

        JSONObject dataObject = jsonObject.getJSONObject("data");

        //上传文件到OSS
        String dir = dataObject.getString("dir");
        String host = dataObject.getString("host");
        String policy = dataObject.getString("policy");

        String sig = dataObject.getString("signature");
        String accessid = dataObject.getString("accessid");
        String securityToken = dataObject.getString("securityToken");

        String callback = dataObject.getString("callback");

        String extension = resourceFilePath.substring(resourceFilePath.lastIndexOf("."));
        String storageKey = dir + UUID.randomUUID() + "." + extension;

        LinkedHashMap<String, String> param = new LinkedHashMap<>();

        param.put("callback", callback);
        //设置动态变量
        param.put("x:userparamuserid", "123456");

        param.put("key", storageKey);//object name
        param.put("OSSAccessKeyId", accessid);
        param.put("policy", policy);
        param.put("Signature", sig);
        param.put("x-oss-security-token", securityToken);

        param.put("x-oss-forbid-overwrite", "true");

        //如果是文件, 则读取文件
        byte[] content = IOUtils.resourceToByteArray(resourceFilePath);

        String uploadResult = uploadFile(host, param, resourceFilePath, content);

        return uploadResult;
    }


    /**
     * 获取Sts Post 方式的Policy.
     */
    public String fetchUploadPolicy() throws IOException {
        String url = host + upload_policy_path;

        HttpPost httpRequest = new HttpPost(url);

        List<NameValuePair> nameValuePairs = new ArrayList<>();
        NameValuePair param1NameValuePair = new BasicNameValuePair("source", "atest");
        NameValuePair param2NameValuePair = new BasicNameValuePair("fileLevel", "privateFile");
        nameValuePairs.add(param1NameValuePair);
        nameValuePairs.add(param2NameValuePair);

        httpRequest.setEntity(new UrlEncodedFormEntity(nameValuePairs, StandardCharsets.UTF_8));

        CloseableHttpResponse httpResponse = httpClient.execute(httpRequest);
        HttpEntity httpEntity = httpResponse.getEntity();

        return EntityUtils.toString(httpEntity);
    }

    /**
     * 上传文件. 仅供参考, 实际中要考虑各种细节.
     */
    public String uploadFile(String url, LinkedHashMap<String, String> map, String fileName, byte[] content) 
            throws IOException {
        MultipartEntityBuilder builder = MultipartEntityBuilder.create();
        ContentType contentType = ContentType.create("application/mp4", StandardCharsets.UTF_8);

        HttpPost httpPost = new HttpPost(url);
        for (String key : map.keySet()) {
            builder.addPart(key, new StringBody(map.get(key), contentType));
            log.debug("formData: " + key + ":" + map.get(key));
        }
        builder.addBinaryBody("file", content, contentType, fileName);

        httpPost.setEntity(builder.build());
        CloseableHttpResponse httpResponse = httpClient.execute(httpPost);
        InputStream ips = httpResponse.getEntity().getContent();
        return IOUtils.toString(ips, StandardCharsets.UTF_8);
    }

    @PostMapping("/upload-callback")
    public ApiResult ossUploadBack(@RequestBody JSONObject callbackBody) throws Exception {
        log.info("[OSS上传callback] content:{}", callbackBody);
        //...校验签名/合法性, 处理结果

        return ApiResult.succeed(UrcResultCode.RESULT_CODE__SUCCESS, "请求签名成功", null);
    }
    
}

```

因为强制客户端设置了callback回调, 所以每次用户上传文件, 服务端都能收到OSS的回调, 我们把相关文件信息记录下来, 就可以做后续的处理了.

我们可以看到在callback里设定了几种变量:
* OSS内置变量, OSS会在上传完成后自动替换变量, 例如 ${contentMd5}
* 服务端自定义变量,例如 sourceId, 在生成Policy时替换 
* 客户端自定义变量, 例如 ${x:userparamuserid}, 尽量用小写, 有些情况下不支持大写.
* callback设置变量有几种方式, 具体可以阅读[官方文档](https://help.aliyun.com/zh/oss/developer-reference/callback){:target="_blank"}


至此, 我们就完成了一个很安全的客户端直传方案, 可以很放心地让客户端直传到OSS上了.


### 参考文档
- [PostObject说明](https://help.aliyun.com/zh/oss/developer-reference/postobject){:target="_blank"}
- [服务器端签名直传](https://help.aliyun.com/zh/oss/use-cases/obtain-signature-information-from-the-server-and-upload-data-to-oss){:target="_blank"}
- [客户端直传安全](https://help.aliyun.com/zh/oss/use-cases/add-signatures-on-the-client-by-using-javascript-and-upload-data-to-oss){:target="_blank"}
- [客户端直传](https://help.aliyun.com/zh/oss/use-cases/uploading-objects-to-oss-directly-from-clients/#36c322a437r3k){:target="_blank"}
- [RAM用户权限设置](https://help.aliyun.com/zh/oss/user-guide/common-examples-of-ram-policies){:target="_blank"}
- [Callback使用说明](https://help.aliyun.com/zh/oss/developer-reference/callback){:target="_blank"}
- [Callback回调处理](https://help.aliyun.com/zh/oss/upload-callbacks-6){:target="_blank"}
- [如何更好地设置文件名](https://help.aliyun.com/zh/oss/use-cases/oss-performance-and-scalability-best-practices){:target="_blank"}
- [STS AssumeRole](https://help.aliyun.com/zh/ram/developer-reference/api-sts-2015-04-01-assumerole){:target="_blank"}
- [RAM Policy Editor](https://help.aliyun.com/zh/oss/developer-reference/ram-policy-editor){:target="_blank"}
- [RAM Policy 列表](https://help.aliyun.com/zh/oss/user-guide/ram-policy-overview/){:target="_blank"}
