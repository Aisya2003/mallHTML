<template>
  <div>
    <el-upload
      action="http://localhost:88/api/file/upload/multiSimpleFile"
      :data="dataObj"
      list-type="picture-card"
      :file-list="fileList"
      :before-upload="beforeUpload"
      :on-remove="handleRemove"
      :on-success="handleUploadSuccess"
      :on-preview="handlePreview"
      :limit="maxCount"
      :on-exceed="handleExceed"
    >
      <i class="el-icon-plus"></i>
    </el-upload>
    <el-dialog :visible.sync="dialogVisible">
      <img width="100%" :src="dialogImageUrl" alt />
    </el-dialog>
  </div>
</template>
<script>
export default {
  name: "multiUpload",
  props: {
    //图片属性数组
    value: Array,
    //最大上传图片数量
    maxCount: {
      type: Number,
      default: 30,
    },
  },
  data() {
    return {
      dataObj: {
        policy: "",
        signature: "",
        key: "",
        ossaccessKeyId: "",
        dir: "",
        host: "",
        uuid: "",
      },
      successCount: 0,
      dialogVisible: false,
      dialogImageUrl: null,
    };
  },
  computed: {
    fileList() {
      let fileList = [];
      if (this.value != null && this.value != undefined) {
        for (let i = 0; i < this.value.length; i++) {
          fileList.push({ url: this.value[i] });
        }
      }

      return fileList;
    },
  },
  mounted() {},
  methods: {
    wait500() {
      return new Promise((resolve) => {
        setTimeout(() => {
          resolve();
        }, 500);
      });
    },
    handleUpload(file) {
      let formData = new FormData(); // 新建一个FormData()对象，这就相当于你新建了一个表单
      formData.append("file", file.file); // 将文件保存到formData对象中
      this.$http({
        url: this.$http.adornUrl("/upload/multiSimpleFile"),
        method: "post",
        data: this.$http.adornData(data, false),
      }).then(({ data }) => {
        console.log(data);
      });
    },
    emitInput(fileList) {
      let value = [];
      for (let i = 0; i < fileList.length; i++) {
        value.push(fileList[i].url);
      }
      console.log("成功了：", value);
      this.$emit("input", value);
    },
    handleRemove(file, fileList) {
      this.emitInput(fileList);
    },
    handlePreview(file) {
      this.dialogVisible = true;
      this.dialogImageUrl = file.url;
    },
    beforeUpload(file) {
      // console.log(file);
    },
    /**
     * 等待所有的图片上传完成
     */
    // async handleUploadSuccess(res, file, fileList) {
    //   console.log(this.successCount,"和",fileList.length)
    //   if (this.successCount >= fileList.length) {
    //     this.fileList.push({
    //       name: file.name,
    //       url: res.data,
    //     });
    //     this.emitInput(this.fileList);
    //   } else {
    //     this.successCount = 0;
    //     for (let i = 0; i < fileList.length; ) {
    //       if (fileList[i].status == "success") {
    //         this.successCount++;
    //         i++;
    //       } else {
    //         await this.wait500();
    //         this.handleUploadSuccess(res, file, fileList);
    //       }
    //     }
    //   }
    // },
    handleUploadSuccess(res, file) {
      this.fileList.push({
        name: file.name,
        // url: this.dataObj.host + "/" + this.dataObj.dir + "/" + file.name； 替换${filename}为真正的文件名
        url: res.data,
      });
      this.emitInput(this.fileList);
    },

    // handleUploadSuccess(res, file, fileList) {
    //   if (res.success) {
    //     if (fileList.every((item) => item.status == "success")) {
    //       fileList.map((item) => {
    //         item.response.success &&
    //           this.fileList.push({
    //             name: item.name,
    //             url: item.response.data,
    //           });
    //       });
    //     }
    //     this.$emit("input", this.fileList);
    //   } else {
    //     this.$message({ message: res.msg, type: "error", duration: 1500 });
    //   }
    // },

    handleExceed(files, fileList) {
      this.$message({
        message: "最多只能上传" + this.maxCount + "张图片",
        type: "warning",
        duration: 1000,
      });
    },
  },
};
</script>
<style>
</style>


