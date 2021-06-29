# x-uploader
>
>分片上传，断点续传
>
>
demo可参考源码中的示例代码

## 版本
- v1.0.2

## 基于
- vue2
- axios
- sparkMD5

## 安装
```
npm install x-uploader-vue
```

## 使用
**在main.js引入**
```
import XUplaoder from 'x-uploader';
```
**在组件中使用**
```
<x-uploader
  :uploadFiles="upFiles"
  :uploadConfig="upConfig"
  :chunkConfig="chunkConfig"
  @handleUpload="handleUpload"
>
    <slot>...</slot>
</x-uploader>
```
### 说明
#### upConfig 上传配置
***
```
upConfig: {
    baseUrl: 'http://localhost', // 上传的根路径
    uploadUrl: {
      headers: { 'Content-Type': 'multipart/form-data' }, // 设置请求头信息
      method: 'post', // 请求方式
      url: '/fileUpload', // 上传的目标路径
      data: {
        token: '1f90kpj7f1cf5s0hiaf1fae1dvk5',
      },  // 上传的参数，会与上传的文件信息合并
    }, // 上传配置信息
    checkUrl: {
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' }, // 设置请求头信息
      method: 'post', // 请求方式
      url: '/checkFileMd5', // 验证的请求地址，断点续传时候，校验已传文件情况
      data: {
        token: '1f90kpj7f1cf5s0hiaf1fae1dvk5',
      }, // 验证的参数
    }, // 断点续传验证地址
    timeout: 1000000, // 单位毫秒
    testTimes: 2, // 上传失败后重试次数
}
```
#### chunkConfig 分片配置
***
```
chunkConfig: {
    size: 5, // 单位M，切片大小
    posts: 5, // 单文件同时发起的请求数，同时并发
}
```
#### uploadFiles 上传的文件
***
```
const fileDom = document.querySelectorAll('input[type=file]')[0];

uploadFiles.push({
    key: `${upFiles.name}-${index}`, // 标识位，需保证唯一
    file: fileDom[0], // 上传的文件
    params: {
      md5: fileKey, // 必传，demo使用的是文件hash，也可以设置其它类型的key，用于断点续传。
    }, // 上传时参数，会和文件参数合并，可手动设置接口所需的参数
    status: 'wait', // wait-待上传 hash-文件hash中 upload-文件上传中 success-上传成功
    progress: 0, // hash时为hash时进度，upload时为上传进度，数值类型，当前百分比比例
});
```
**上传参数，保留字段。其余可在uploadFiles里的params设置**
```
chunks=15 // 分片数量
chunk=1 // 当前分片
chunkSize=100 // 分片大小
file=binary // 分片的文件
```
#### handleUpload(data) 上传状态的emit方法
***
check-验证断点时返回，会在第一次上传前触发
```
{
    type: 'check',  
    data: {
        fileKey： uploadFiles里的key，用于标识当前哪个文件
        status: checked 接口请求成功 /checkErr 接口请求失败
        result: 接口返回的结果，如果断点，具体的续传信息这里返回
    }
}
```
upload-上传时返回 
```
{
    type: 'upload',  
    data: {
        fileKey： uploadFiles里的key，用于标识当前哪个文件
        status: upload 上传成功，分片还没上传中/uploadErr 分片上传失败/success 上传成功并完成
        progress: 上传的进度，百分比比例，数字类型,例如：10，实际为进度的10%
        result: 接口返回的结果
    }
}
```
status-状态更新时返回
```
{
    type: 'status',  
    data: {
        fileKey： uploadFiles里的key，用于标识当前哪个文件
        status: stop 上传停止,触发upConfig里的testTimes时候，会返回
                check 上传文件接口验证中，主要验证断点信息，使用uploadFiles的params的md5参数
    }
}
```
notfound-文件找不到时返回
alreadyUpload-文件已上传
```
{
    type: 'notfound/alreadyUpload',  
    data: {
        fileKey： uploadFiles里的key，用于标识当前哪个文件
    }
}
```
#### getFileHash()文件hash的方法，仅示例，可以自行封装，非必须
```
getFileHash(index, upFile) {
  const that = this;
  return new Promise((resolve, reject) => {
    const blobSlice = File.prototype.slice;
    const chunkSize = this.chunkConfig.size * 1024 * 1024;
    const chunks = Math.ceil(upFile.size / chunkSize);
    const sparkIns = new SparkMD5.ArrayBuffer();
    const fileReader = new FileReader();
    let curChunk = 0;

    function getHashProgress() {
      let curProgress = 0;
      let curStatus = 'upload';
      if (curChunk < chunks) {
        curStatus = 'hash';
        curProgress = ((curChunk / chunks) * 100).toFixed(2);
      }
      that.upFiles.splice(index, 1, {
        ...that.upFiles[index],
        status: curStatus,
        progress: curProgress,
      });
    }
    function nextChunk() {
      const start = curChunk * chunkSize;
      const end = (start + chunkSize) >= upFile.size ? upFile.size : (start + chunkSize);
      fileReader.readAsArrayBuffer(blobSlice.call(upFile, start, end));
      getHashProgress(index, curChunk, chunks);
    }
    fileReader.onload = (e) => {
      sparkIns.append(e.target.result);
      curChunk += 1;
      if (curChunk < chunks) {
        nextChunk();
      } else {
        getHashProgress(index, curChunk, chunks);
        const result = sparkIns.end();
        resolve(result);
      }
    };
    fileReader.onerror = (err) => {
      reject(err);
    };
    nextChunk();
  });
},
```

