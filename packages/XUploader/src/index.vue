<template>
  <div>
    <slot></slot>
  </div>
</template>

<script>
import Axios from 'axios';

export default {
  name: 'x-uploader',
  props: {
    uploadConfig: Object, // 上传设置
    chunkConfig: Object, // 分片设置
    uploadFiles: Array, // 上传的文件列表
  },
  watch: {
    uploadFiles() {
      this.initQueue();
    },
  },
  computed: {
    // 分片大小
    chunkSize() {
      return 1024 * 1024 * this.chunkConfig.size;
    },
  },
  data() {
    return {
      axiosIns: null,
      upFiles: [],
      upQueue: {},
      postsNumber: 0,
      errFileKeys: [],
    };
  },
  methods: {
    initQueue() {
      const curQueue = [];
      this.uploadFiles.forEach((upFile) => {
        if (upFile.file) {
          curQueue.push(upFile.key);
          if (!this.upQueue[upFile.key]) {
            const chunkNum = Math.ceil(upFile.file.size / this.chunkSize);
            const waitUpload = [];
            for (let i = 0; i < chunkNum; i += 1) {
              waitUpload.push(i);
            }
            this.upQueue[upFile.key] = {
              max: chunkNum,
              wait: waitUpload,
              success: [],
            };
          }
        }
      });
      // 去除无用队列数据
      Object.keys(this.upQueue).forEach((fileKey) => {
        if (!curQueue.includes(fileKey)) {
          this.deleteQueue(fileKey);
        }
      });
      if (curQueue.length > 0) {
        this.prepareUpload();
      }
    },
    deleteQueue(fileKey) {
      delete this.upQueue[fileKey];
    },
    /**
     * 状态为 wait 且 不存在文件 hash 值的时候，请求接口，验证文件状态，并完成断点续传。验证失败，则直接进入重新上传
     */
    checkFile(fileKey, upFile) {
      if (upFile.status === 'wait') {
        this.emitTo('status', {
          fileKey,
          status: 'check',
        });
        Object.assign(
          this.uploadConfig.checkUrl.data,
          upFile.params,
          { chunkSize: this.chunkSize },
        );
        this.getAxios().request(this.uploadConfig.checkUrl).then((result) => {
          this.emitTo('check', {
            fileKey,
            status: 'checked',
            result: result.data,
          });
        }).catch((err) => {
          // 验证失败，则直接进入重新上传
          this.emitTo('check', {
            fileKey,
            status: 'checkErr',
            result: err,
          });
        });
      }
    },
    /**
     * Axios实例
     */
    getAxios() {
      if (!this.axiosIns) {
        this.axiosIns = Axios.create({
          baseURL: this.uploadConfig.baseUrl,
          timeout: this.uploadConfig.timeout,
        });
        this.axiosIns.interceptors.request.use((config) => {
          const configCopy = config;
          if (configCopy.method.toUpperCase() === 'POST') {
            const formData = new FormData();
            Object.keys(configCopy.data).forEach((key) => {
              formData.append(key.toString(), configCopy.data[key]);
            });
            configCopy.data = formData;
            configCopy.headers['Content-Type'] = 'application/x-www-form-urlencoded';
          } else if (configCopy.method.toUpperCase() === 'FILE') {
            configCopy.headers['Content-Type'] = 'multipart/form-data';
            configCopy.method = 'POST';
          }
          return configCopy;
        });
      }
      return this.axiosIns;
    },
    /**
     * 组装上传请求的参数
     */
    getParams(params, upFile, chunks, chunk) {
      const nextSize = Math.min((chunk + 1) * this.chunkSize, upFile.size);
      const fileData = upFile.slice(chunk * this.chunkSize, nextSize);
      return Object.assign(params, {
        chunks,
        chunk,
        chunkSize: this.chunkSize,
        file: fileData,
      });
    },
    /**
     * 获取队列中上传的文件
     */
    getFile(fileKey) {
      let backData = '';
      this.uploadFiles.forEach((file) => {
        if (file.key === fileKey) {
          backData = file;
          return false;
        }
        return true;
      });
      return backData;
    },
    /**
     * 准备发起上传请求
     * 状态 stop 和 check 不参与上传
     */
    prepareUpload() {
      Object.keys(this.upQueue).forEach((fileKey) => {
        const curFile = this.getFile(fileKey);
        let deleteStatus = '';
        if (curFile) {
          switch (curFile.status) {
            case 'stop':
            case 'check':
            case 'hash':
              break;
            case 'success':
              deleteStatus = 'alreadyUpload';
              break;
            case 'continue':
              if (Array.isArray(curFile.waiting) && curFile.waiting.length > 0) {
                curFile.waiting = (curFile.waiting).map((item) => parseInt(item, 10));
                this.upQueue[fileKey].wait.forEach((chunkIndex, index) => {
                  if (!(curFile.waiting).includes(chunkIndex)) {
                    this.upQueue[fileKey].wait.splice(index, 1);
                    this.upQueue[fileKey].success.push(chunkIndex);
                  }
                });
              }
              this.emitTo('upload', {
                fileKey,
                status: 'upload',
                progress: this.getProgress(
                  (this.upQueue[fileKey].success).length,
                  this.upQueue[fileKey].max,
                ),
              });
              break;
            case 'upload':
              if (this.upQueue[fileKey].wait.length > 0) {
                this.upQueue[fileKey].wait.forEach((chunkIndex, index) => {
                  if (this.postsNumber < this.chunkConfig.posts) {
                    this.postsNumber += 1;
                    const params = this.getParams(
                      curFile.params,
                      curFile.file,
                      this.upQueue[fileKey].max,
                      chunkIndex,
                    );
                    // 删除wait里的
                    this.upQueue[fileKey].wait.splice(index, 1);
                    this.doUpload(fileKey, chunkIndex, params);
                    return true;
                  }
                  return false;
                });
              } else if (this.errFileKeys.length >= this.uploadConfig.testTimes) {
                this.emitTo('status', {
                  fileKey,
                  status: 'stop',
                });
              }
              break;
            case 'wait':
            default:
              this.checkFile(fileKey, curFile);
              break;
          }
        } else {
          deleteStatus = 'notfound';
        }
        // 如果异常
        if (deleteStatus) {
          this.emitTo(deleteStatus, {
            fileKey,
          });
          this.deleteQueue(fileKey);
        }
      });
    },
    getProgress(successLength, maxLength) {
      return ((successLength / maxLength) * 100).toFixed(2);
    },
    /**
     * 传输完成，并发数量减1
     * 传输成功，1、减少 wait 队列，2、更新进度
     * @param fileKey
     * @param chunkIndex
     * @param params
     */
    doUpload(fileKey, chunkIndex, params) {
      Object.assign(this.uploadConfig.uploadUrl.data, params);
      this.uploadConfig.uploadUrl.timeout = this.uploadConfig.timeout;
      this.getAxios().request(this.uploadConfig.uploadUrl).then((result) => {
        this.postsNumber -= 1;
        if (this.upQueue && this.upQueue[fileKey]) {
          this.upQueue[fileKey].success.push(chunkIndex);
          const successLength = this.upQueue[fileKey].success.length;
          let curProgress = 100;
          let curStatus = 'success';
          if (successLength < this.upQueue[fileKey].max) {
            curStatus = 'upload';
            curProgress = this.getProgress(successLength, this.upQueue[fileKey].max);
          }
          if (this.errFileKeys.length > 0) {
            const errIndex = this.errFileKeys.findIndex(fileKey);
            if (errIndex > -1) {
              this.errFileKeys.splice(errIndex, 1);
            }
          }
          this.emitTo('upload', {
            fileKey,
            status: curStatus,
            progress: curProgress,
            result,
          });
        } else {
          this.emitTo('upload', {
            fileKey,
            status: 'uploadErr',
            result: 'file not exists',
          });
        }
      }).catch((err) => {
        console.log(err);
        this.errFileKeys.push(fileKey);
        this.initQueue();
        this.emitTo('upload', {
          fileKey,
          status: 'uploadErr',
          result: err,
        });
      });
    },
    /**
     * emit数据
     * @param type
     * @param data
     */
    emitTo(type, data = '') {
      this.$emit('handleUpload', {
        type,
        data,
      });
    },
  },
};
</script>
