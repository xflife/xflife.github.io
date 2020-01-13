---
title: 阿里云上传--web 通过 STS 授权直传
date: 2018-09-09
tags: docs
categories: 文档
---

**目的**是让'前端统治世界'，哈哈哈哈！！！人家开个玩笑啦啦啦。其实目前上传有以下缺点：

- **第一** ：上传慢。先上传到应用服务器，再上传到OSS，网络传送多了一倍。如果数据直传到OSS，不走应用服务器，速度将大大提升，而且OSS是采用BGP带宽，能保证各地各运营商的速度；
- **第二** ：扩展性不好。如果后续用户多了，应用服务器会成为瓶颈；
- **深度整合** ：费用高。由于OSS上传流量是免费的。如果数据直传到OSS，不走应用服务器，那么将能省下几台应用服务器。
<!-- more -->
-------------------
### 阿里云直传流程
**1**：请求 STS 授权，并且缓存其返回信息；
**2**：计算文件 MD5 ；
**3**：构造阿里云直传需要的参数；
**4**：进行判断图片在库是否存在；如果存在请看 `5`，如果不存在请看 `6`；
**5**：资源存在，如果是`图片`可以请求阿里云获得图片资源信息；
**6**：开始上传，上传成功后，如果是`图片`可以请求阿里云获得图片资源信息；
**7**：上传失败,判断是否是 `403 Forbidden`错误，如果是，返回 1；其他错误，自己处理。

## 废话不多说，上代码：

### 得到文件名后缀

``` javascript
const getSuffix = (filename)=> {
  let pos = filename.lastIndexOf('.')
  let suffix = ''
  if (pos != -1) {
    suffix = filename.substring(pos)
  }
  return suffix;
}
```
### 得到图片 MD5 方法

``` javascript
const calculateImageMD5 = (params, handleCb)=> {
  let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice,
    file = params.file,
    chunkSize = 2097152,                             // Read in chunks of 2MB
    chunks = Math.ceil(file.size / chunkSize),
    currentChunk = 0,
    spark = new SparkMD5.ArrayBuffer(),
    fileReader = new FileReader();
  fileReader.onload = function (e) {
    console.log('read chunk nr', currentChunk + 1, 'of', chunks);
    spark.append(e.target.result);                   // Append array buffer
    currentChunk++;

    if (currentChunk < chunks) {
      loadNext();
    } else {
      console.log('finished loading');
      let filename = '/' + spark.end() + getSuffix(params.file.name);
      console.info('computed hash', filename);  // Compute hash

      console.warn('文件 MD5计算完成，开始阿里云上传',new Date());
      //开始阿里云上传
      uploadForALiYun(params, filename, handleCb);
    }
  };

  fileReader.onerror = function () {
    console.warn('oops, something went wrong.');
  };

  function loadNext() {
    var start = currentChunk * chunkSize,
      end = ((start + chunkSize) >= file.size) ? file.size : start + chunkSize;

    fileReader.readAsArrayBuffer(blobSlice.call(file, start, end));
  }

  loadNext();
}
```
### STS 请求 伪代码
```javascript
 const stsReuest =>(params) {
  let defer = Q.defer();
  if(!window.STSOBJEACT){
    HttpClient.post('你的STS授权接口', {url: '阿里云图片域名', dir: '阿里云图片目录' })
      .then((res) => {
        window.STSOBJEACT = res;
        //计算图片  MD5
        calculateImageMD5(params,handleCb);
        defer.resolve(res);
      }, (err)=> {
        defer.reject(err);
      });
  }else{
    calculateImageMD5(params,handleCb);
  }
  return defer.promise;
}
```

### 阿里云直传 伪代码 -- 包含判断图片是否重复
```javascript
function uploadForALiYun(param, filename, handleCb) {
  let defer = Q.defer();
  let params = {
    name: 'file',
    file: param.file,
    fields: [
      {name: 'key', value: window.STSOBJEACT.dir + filename},
      {name: 'policy', value: window.STSOBJEACT.policy},
      {name: 'OSSAccessKeyId', value: window.STSOBJEACT.accessid},
      {name: 'success_action_status', value: 200},
      {name: 'signature', value: window.STSOBJEACT.signature},
    ]
  }
  let imageALiPath = 'http://' + utils.aLiUploadUrl + '/' + utils.aLiUploadDir + filename;
  //判断图片存在与否
  HttpClient.head(imageALiPath).then(()=>{
    //直接得到图片信息
    getImageInfo(imageALiPath,handleCb);
  },()=>{
    //上传图片
    HttpClient.upload('http://' + utils.aLiUploadUrl, params)
      .then(() => {
        getImageInfo(imageALiPath,handleCb);
        defer.resolve();
      }, (err) => {
        //没有权限
        if (err.code == 403) {
          window.STSOBJEACT = null;
          uploadImageApi(params, handleCb);
        }
        defer.reject(err);
      });
  });
  return defer.promise;
}
```
> **注意：**如果检测`图片重复`接口出现`404 Not Found`,可能 HTTP Cache 机制导致的，建议直接加个标识:
let imageALiPath = 'http://' + utils.aLiUploadUrl + '/' + utils.aLiUploadDir + filename ` + '?(time=)'+new Date().getTime();`


### 得到图片信息，并且返回
```javascript
function getImageInfo(imageALiPath,handleCb){
  let defer = Q.defer();
  console.warn('阿里云上传完成，开始获取图片信息', new Date());
  //获取图片地址
  HttpClient.get(imageALiPath + '@infoexif')
    .then((result) => {
      let image_result_object = {
        "data": {
          "image_rotation": 0.0,
          "image_tags": [],
          "imageHwlv": 0.0,
          "image_padding_right": 0.0,
          "image_primary_color": "",
          "image_height": 0.0,
          "image_padding_top": 0.0,
          "image_date": '',
          "image_id": "0",
          "image_orientation": 0,
          "image_start_point_x": 0.0,
          "image_flip_vertical": 0,
          "imageWhlv": 0.0,
          "image_start_point_y": 0.0,
          "image_remark": "",
          "image_scale_vertical": 0.0,
          "image_padding_left": 0.0,
          "image_url": "",
          "image_padding_bottom": 0.0,
          "image_width": 0.0,
          "image_scale": 0.0,
          "image_flip_horizontal": 0,
          "image_content": ""
        },
        "error_code": 10000,
        "info": "接口响应正常！"
      };
      image_result_object.data.image_orientation = result.Orientation && result.Orientation.value || '0';
      image_result_object.data.image_width = result.ImageWidth.value;
      image_result_object.data.image_height = result.ImageHeight.value;
      image_result_object.data.image_url = imageALiPath;
      image_result_object.data.image_date = result.DateTimeOriginal && new Date(result.DateTimeOriginal.value.split(" ")[0])
          .getTime() / 1000 || new Date().getTime() / 1000;
      console.log('image_result_object', image_result_object);
      //处理成功回调
      handleCb && handleCb.handleSuccess(Object.assign({}, image_result_object));
      defer.resolve(Object.assign({}, image_result_object));
    }, (err) => {
      handleCb && handleCb.handleError(err);
      defer.reject(err);
    });
  return defer.promise;
}
```



> **注意：**如果请求`上传`接口出现`400 Bad Request`,可能是参数顺序有误，可以参考下图参数顺序。

![示例上传参数顺序](http://static.timeface.cn/webupload/demo.png)


## 反馈与建议
- 微信：其实我是不会告诉你的！！！
- 邮箱：<xf_life@yeah.net>

---------
感谢阅读这份帮助文档。

