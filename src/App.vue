<template>
  <div id="app">
    <x-uploader
      :uploadFiles="upFiles"
      :uploadConfig="upConfig"
      :chunkConfig="chunkConfig"
      @handleUpload="handleUpload"
    >
      <div class="upload-area" :key="index" v-for="(upFile, index) in upFiles">
        <div class="progress-base">
          <div class="progress-line" :style="{ width: upFile.progress + '%' }"></div>
        </div>
        <div>上传文件：{{ upFile.file ? upFile.file.name : '' }} 当前进度：{{ upFile.status }}</div>
        <div>
          <input type="file" @change="handleChange(index)" />
        </div>
        <button
          @click="doPause(index)"
          v-if="upFile.status === 'upload' || upFile.status === 'stop'"
        >
          {{ upFile.status === 'stop' ? '继续' : '暂停' }}
        </button>
        <button @click="doUpload(index)">上传</button>
      </div>
    </x-uploader>
  </div>
</template>

<script>
import SparkMD5 from 'spark-md5';

export default {
  name: 'App',
  data() {
    return {
      upConfig: {
        baseUrl: 'http://localhost:8082/api',
        uploadUrl: {
          headers: { 'Content-Type': 'multipart/form-data' },
          method: 'post',
          url: '/fileUpload',
          data: {
            userId: 4378,
            projectId: 2,
          },
        }, // 上传地址
        checkUrl: {
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
          method: 'post',
          url: '/checkFileMd5', // 断点续传时候，校验已传文件情况
          data: {
            userId: 4378,
            projectId: 2,
          },
        }, // 断点续传验证地址
        timeout: 1000000, // 单位毫秒
        testTimes: 2, // 错误重试次数
      },
      chunkConfig: {
        size: 5, // 单位M，切片大小
        posts: 5, // 单文件同时发起的请求数，同时并发
      },
      upFiles: [],
    };
  },
  created() {
    this.initFilers(1);
  },
  methods: {
    initFilers(numbers) {
      for (let i = 0; i < numbers; i += 1) {
        this.upFiles.push({
          status: 'wait',
          progress: 0,
        });
      }
    },
    /**
     * 暂停上传
     */
    doPause(index) {
      let curStatus = '';
      if (this.upFiles[index].status === 'stop') {
        curStatus = 'upload';
      } else {
        curStatus = 'stop';
      }
      this.upFiles.splice(index, 1, {
        ...this.upFiles[index],
        status: curStatus,
      });
    },
    /**
     * 上传分片
     */
    doUpload(index) {
      const fileDom = document.querySelectorAll('input[type=file]')[index];
      if (fileDom.value) {
        this.getFileKey(index, fileDom.files[0]).then((result) => {
          this.createUploadFiles(index, fileDom.files[0], result);
        }).catch((err) => {
          console.log(err);
        });
      }
    },
    createUploadFiles(index, upFile, fileKey) {
      this.upFiles.splice(index, 1, {
        key: `${upFile.name}-${index}`, // 需保证唯一
        file: upFile,
        params: {
          md5: fileKey, // 文件hash
          fileName: upFile.name,
          type: upFile.type,
          size: upFile.size,
          lastModifiedDate: upFile.lastModifiedDate,
        },
        status: 'wait',
        progress: 0,
      });
    },
    /**
     * 获取文件唯一key，用于后端判断是否已经有上传，断点续传使用
     */
    getFileKey(index, upFile) {
      return this.getFileHash(index, upFile);
    },
    /**
     * 上传文件hash
     */
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
    /**
     * 获取文件的索引
     * @param fileKey
     * @return {number}
     */
    getFileIndex(fileKey) {
      let fileIndex = -1;
      this.upFiles.forEach((upFile, index) => {
        if (upFile.key === fileKey) {
          fileIndex = index;
          return false;
        }
        return true;
      });
      return fileIndex;
    },
    handleChange(fileIndex) {
      if (this.upFiles[fileIndex].file) {
        this.upFiles.splice(fileIndex, 1, {
          status: 'wait',
          progress: 0,
        });
      }
    },
    handleUpload(data) {
      let emitData = data.data || {};
      const fileIndex = this.getFileIndex(emitData.fileKey);
      let emitType = data.type;
      if (fileIndex > -1) {
        if (emitType === 'alreadyUpload') {
          emitType = 'status';
          emitData = {
            status: 'success',
          };
        }
        switch (emitType) {
          case 'check':
            switch (emitData.status) {
              case 'checked':
                switch (emitData.result.data.status.toString()) {
                  case '0': // 已上传
                    this.upFiles.splice(fileIndex, 1, {
                      ...this.upFiles[fileIndex],
                      status: 'success',
                      progress: 100,
                    });
                    break;
                  case '2': // 续传
                    this.upFiles.splice(fileIndex, 1, {
                      ...this.upFiles[fileIndex],
                      status: 'continue',
                      progress: 0,
                      waiting: emitData.result.data.data,
                    });
                    break;
                  case '1': // 未上传
                  default:
                    this.upFiles.splice(fileIndex, 1, {
                      ...this.upFiles[fileIndex],
                      status: 'upload',
                      progress: 0,
                    });
                    break;
                }
                break;
              case 'checkErr':
              default:
                console.log(emitData.result);
                break;
            }
            break;
          case 'upload':
            switch (emitData.status) {
              case 'upload':
                if (this.upFiles[fileIndex].status !== 'stop') {
                  this.upFiles.splice(fileIndex, 1, {
                    ...this.upFiles[fileIndex],
                    status: emitData.status,
                    progress: emitData.progress || 0,
                  });
                }
                break;
              case 'success':
                this.upFiles.splice(fileIndex, 1, {
                  status: 'success',
                  progress: 100,
                });
                break;
              case 'uploadErr':
              default:
                break;
            }
            break;
          case 'status':
            switch (emitData.status) {
              case 'stop':
                this.upFiles.splice(fileIndex, 1, {
                  ...this.upFiles[fileIndex],
                  status: 'stop',
                });
                break;
              case 'check':
                this.upFiles[fileIndex].status = emitData.status;
                break;
              default:
                break;
            }
            break;
          case 'notfound':
          default:
            this.splice(fileIndex, 1);
            break;
        }
      }
    },
  },
};
</script>

<style scoped>
  .upload-area {
    margin-bottom: 30px;
  }
  .progress-base {
    height: 20px;
    width: 500px;
    position: relative;
    background-color: #eeeeee;
    border-radius: 4px;
    margin-bottom: 10px;
  }
  .progress-line {
    height: 20px;
    position: absolute;
    top: 0;
    left: 0;
    background-color: seagreen;
    border-radius: 4px;
    width: 0px;
  }
</style>
