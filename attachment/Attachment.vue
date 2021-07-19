<template>
  <div class="dsi-attachment">
    <div class="dsi-attachment__container">
      <dsi-dropzone
        :read-only="readOnly"
        :configs="configsFile"
        :files="fileList"
        @change="onChange"
      >
        <template slot-scope="{ isEnter }">
          <dsi-filezone
            ref="file-zone"
            :read-only="readOnly"
            :configs="configsFile"
            :files="fileList"
            :is-enter="isEnter"
            :options="fileCategories"
            :no-progress="noProgress"
            @sort="sortFiles"
            @remove="handleRemove"
            @updateOrder="updateOrder"
            @beforeDownload="$emit('beforeDownload')"
            @afterDownload="$emit('afterDownload')"
          />
        </template>
      </dsi-dropzone>
    </div>
  </div>
</template>

<script>
import _ from "lodash";
import DsiDropzone from "./DsiDropzone";
import DsiFilezone from "./DsiFilezone";

function genTransId() {
  let s4 = () => {
    return Math.floor((1 + Math.random()) * 0x10000)
      .toString(16)
      .substring(1);
  };
  return s4() + s4() + "-" + s4() + s4();
}

export default {
  name: "DsiAttachment",
  components: {
    DsiDropzone,
    DsiFilezone,
  },
  props: {
    baseURL: {
      type: String,
      default: "",
    },
    files: {
      type: Array,
      default: () => [],
    },
    configs: {
      type: Object,
      default() {
        return this.configDefault;
      },
    },
    compId: {
      type: String,
      default: "",
    },
    menuId: {
      type: String,
      default: "",
    },
    noProgress: {
      type: Boolean,
      default: false,
    },
    readOnly: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      selectedFiles: [],
      fileList: [],
      fileCategories: [],
      configDefault: {
        sortable: true,
        multiple: true,
        // unit Bytes
        maxFileSize: 100000000,
        chunkSize: 900000,
        maxSizeTotal: 9800000000,
        acceptedFiles: [],
        maxFiles: 5,
        placeholder: "Click or drop files here...",
        groupId: this.generateUUID(),
      }
    };
  },
  computed: {
    configsFile() {
      return {...this.configDefault, ...this.configs};
    },
    dropStyle() {
      return this.isEnter
        ? "background-color : rgba(149,149,149,0.8) !important;"
        : "";
    },
  },
  watch: {
    files(val) {
      if (Array.isArray(val) && val.length) {
        val.map((e) => {
          e.transId = e.transId || genTransId();
          e.selected = false;
          e.isFileServer = true;
          if(!(e instanceof Blob)) {
            e.name = e.fileName;
            e.size = e.fileSize;
            e.uuid = e.fileId;
          }
        });
        this.fileList = _.uniqBy([...this.fileList, ...val],"uuid");
      }
    },
    fileList(val) {
      this.$emit("change", val);
    }
  },
  async created() {
    await this.getListCategory();
  },
  methods: {
    clearFile() {
      this.fileList = [];
    },
    generateUUID() {
      let d = new Date().getTime();
      if (
        typeof performance !== "undefined" &&
        typeof performance.now === "function"
      ) {
        d += performance.now(); //use high-precision timer if available
      }
      return "xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx".replace(
        /[xy]/g,
        function (c) {
          var r = (d + Math.random() * 16) % 16 | 0;
          d = Math.floor(d / 16);
          return (c === "x" ? r : (r & 0x3) | 0x8).toString(16);
        }
      );
    },
    sortFiles: _.debounce(function (files) {
      this.fileList = [...files];
      this.updateOrder(this.fileList);
    }, 500),

    updateOrder(files) {
      let arrSort = _.compact(_.map(files, "uuid"));
      this.$http
        .put("/rest/v2/attachment-file/update/order", arrSort)
        .then(() => {
          // this.$notify(this.$msg("message.saved", "Saved"));
          this.$emit("on-done", this.configsFile.groupId);
        });
    },
    handleRemove(file) {
      const index = this.fileList.findIndex(
        (item) => item.transId === file.transId
      );

      if (index < 0) {
        return;
      }
      if (file.uuid) {
        this.deleteFile(file.uuid);
      }
      this.fileList.splice(index, 1);
    },

    deleteFile(uuid) {
      this.$http
        .delete("/rest/v2/attachment-file/delete/file/" + uuid)
        .then(() => {
          //todo remove file
        })
        .catch(() => {
          //todo show error notification
        });
    },

    onChange(type, value) {
      if (type === "add") {
        const items = _.map(value, (item) => {
          item.createUserName = this.$store.getters.userName;
          item.fileCategoryId = null;
          item.createDate = new Date();
          return item;
        });
        this.fileList = [...this.fileList, ...items];
      }
    },
    async getListCategory() {
      try {
        const response = await this.$http.get("/rest/v2/fileservice/category");
        this.fileCategories = response.data;
        this.fileCategories.map((e) => {
          e.text = e.fileCategoryName;
          e.value = e.fileCategoryId;
        });
      } catch (error) {
        this.fileCategories = [];
        console.log("ERROR: Occur when it tries to get file category", error);
      }
    },
  },
};
</script>

<style lang="scss" scoped>

</style>
