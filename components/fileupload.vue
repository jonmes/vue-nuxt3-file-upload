<script setup>
import { useField } from "vee-validate";

const emit = defineEmits([
  "update:modelValue",
  "itemChange",
  "select",
  "clear",
]);

const props = defineProps({
  modelValue: {
    type: [Array, Object, String],
  },
  init: {
    type: [Array, Object, String],
  },
  allowedTypes: {
    type: [Array, String],
    default: "/*",
  },
  accept: {
    type: String,
    default: null,
  },
  disabled: {
    type: Boolean,
    default: false,
  },
  invalidFileSizeMessage: {
    type: String,
    default: "{0}: Invalid file size, file size should be smaller than {1}.",
  },
  invalidFileTypeMessage: {
    type: String,
    default: "{0}: Invalid file type, allowed file types: {1}.",
  },
  invalidFileLimitMessage: {
    type: String,
    default: "Maximum number of files exceeded, limit is {0} at most.",
  },
  maxFileSize: {
    type: Number,
    required: true,
  },
  inputClass: {
    type: String,
    default: "",
    required: false,
  },
  name: {
    type: String,
    default: undefined,
    required: true,
  },
  auto: {
    type: Boolean,
    default: false,
  },
  rules: {
    type: String,
    default: "",
    required: false,
  },
  hideDetail: {
    type: Boolean,
    default: false,
    required: false,
  },
  prependIcon: Function,
  fileLimit: { type: Number, default: 1 },
  text: { type: String },
  fileTypes: { type: String },
  thumbnail: { type: Boolean, default: false },
  customizableText: { type: Boolean, default: false },
  preview: {
    type: Boolean,
    default: false,
  },
  editImage: String,
  previewSrc: [Array, String],
  wrapperClass: String,
});

const {
  errorMessage,
  value: inputValue,
  meta,
} = useField(props.name, props.rules, {
  initialValue: props.modelValue,
});

const fileInput = ref(null);
const files = ref([]);
const uploadedFileCount = ref(0);
const uploadedFiles = ref(0);
const rawFiles = ref([]);
const messages = ref([]);
const dataUrls = ref([]);
const isIcon = ref(true);
const previewSrc = ref(props.init);
const selectedType = ref([]);
const focused = ref(false);
const isUploadDone = ref(false);
const base64Files = ref([]);

onMounted(() => {
  let i = 0;
  const urlToObject = async (imageUrl) => {
    let filename = imageUrl.split("?")[0].split("/").slice(-1)[0].split(".")[0];
    fetch(imageUrl)
      .then((response) => response.blob())
      .then((blob) => {
        // Create a File object from the Blob
        const file = new File([blob], filename, { type: blob.type });
        file.objectURL = URL.createObjectURL(file);
        createImage(file);
        files.value.push(file);
      });
  };
  while (i < previewSrc.value?.length) {
    urlToObject(previewSrc.value[i]);
    i++;
  }
});

const formatBytes = (bytes, decimals = 2) => {
  if (bytes === 0) return "0 Bytes";

  const k = 1024;
  const dm = decimals < 0 ? 0 : decimals;
  const sizes = ["Bytes", "KB", "MB", "GB", "TB", "PB", "EB", "ZB", "YB"];

  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return parseFloat((bytes / Math.pow(k, i)).toFixed(dm)) + " " + sizes[i];
};

const open = () => {
  fileInput.value.click();
};

const onFileSelect = async (event) => {
  files.value = files.value || [];
  let _files = event.dataTransfer
    ? event.dataTransfer.files
    : event.target.files;

  for (let file of _files) {
    if (!isFileSelected(file)) {
      if (validate(file)) {
        if (isImage(file)) {
          file.objectURL = window.URL.createObjectURL(file);
        }
        createImage(file);
        files.value.push(file);
      } else {
        console.log("the else", messages.value);
      }
    }
  }

  emit("select", { originalEvent: event, files: files.value });

  if (props.fileLimit) {
    checkFileLimit();
  }

  if (props.auto && hasFiles.value && !isFileLimitExceeded()) {
    upload();
  }
};

function isFileSelected(file) {
  if (files.value && files.value.length) {
    for (let sFile of files.value) {
      if (
        sFile.name + sFile.type + sFile.size ===
        file.name + file.type + file.size
      )
        return true;
    }
  }

  return false;
}

function validate(file) {
  if (props.accept && !isFileTypeValid(file)) {
    messages.value == null ? (messages.value = []) : messages.value;
    messages.value.push(
      props.invalidFileTypeMessage
        .replace("{0}", file.name)
        .replace("{1}", props.accept)
    );

    return false;
  }

  if (props.maxFileSize && file.size > props.maxFileSize) {
    messages.value == null ? (messages.value = []) : messages.value;
    messages.value.push(
      props.invalidFileSizeMessage
        .replace("{0}", file.name)
        .replace("{1}", formatSize(props.maxFileSize))
    );

    return false;
  }

  return true;
}

function isFileTypeValid(file) {
  let acceptableTypes = props.accept.split(",").map((type) => type.trim());

  for (let type of acceptableTypes) {
    let acceptable = isWildcard(type)
      ? getTypeClass(file.type) === getTypeClass(type)
      : file.type == type ||
        getFileExtension(file).toLowerCase() === type.toLowerCase();

    if (acceptable) {
      return true;
    }
  }

  return false;
}

function isWildcard(fileType) {
  return fileType.indexOf("*") !== -1;
}

function getTypeClass(fileType) {
  return fileType.substring(0, fileType.indexOf("/"));
}

function getFileExtension(file) {
  return "." + file.name.split(".").pop();
}
function isImage(file) {
  return /^image\//.test(file.type);
}
function formatSize(bytes) {
  if (bytes === 0) {
    return "0 B";
  }

  let k = 1000,
    dm = 1,
    sizes = ["B", "KB", "MB", "GB", "TB", "PB", "EB", "ZB", "YB"],
    i = Math.floor(Math.log(bytes) / Math.log(k));

  return parseFloat((bytes / Math.pow(k, i)).toFixed(dm)) + " " + sizes[i];
}

function checkFileLimit() {
  if (isFileLimitExceeded()) {
    messages.value == null ? (messages.value = []) : messages.value;
    messages.value.push(
      props.invalidFileLimitMessage.replace("{0}", props.fileLimit.toString())
    );
  }
}

function isFileLimitExceeded() {
  if (
    props.fileLimit &&
    props.fileLimit <= files.value.length + uploadedFileCount.value &&
    focused.value
  ) {
    focused.value = false;
  }

  return (
    props.fileLimit &&
    props.fileLimit < files.value.length + uploadedFileCount.value
  );
}

function onMessageClose() {
  messages.value = null;
}

const hasFiles = computed(() => {
  return files.value && files.value.length > 0;
});

const chooseDisabled = computed(() => {
  return (
    props.disabled ||
    (props.fileLimit &&
      props.fileLimit <= files.value.length + uploadedFileCount.value)
  );
});
const uploadDisabled = computed(() => {
  return (
    props.disabled ||
    !hasFiles.value ||
    (props.fileLimit && props.fileLimit < files.value.length)
  );
});
const cancelDisabled = computed(() => {
  return props.disabled || !hasFiles.value;
});

function clear() {
  files.value = [];
  messages.value = null;
  base64Files.value = [];
  emit("clear");
}

function clearInputElement() {
  fileInput.value = "";
}

function remove(index) {
  files.value.splice(index, 1);
  base64Files.value = [];
  for (let file of files.value) {
    createImage(file);
  }
  //   emits("update:modelValue", a);
}

const readFile = (file) => {
  return new Promise(function (resolve, reject) {
    const fr = new FileReader();

    fr.onload = function () {
      resolve(fr.result);
    };

    fr.onerror = function () {
      reject(fr);
    };

    fr.readAsDataURL(file);
  });
};

function createImage(file) {
  var reader = new FileReader();
  reader.onload = (e) => {
    base64Files.value.push(e.target.result);
  };
  reader.readAsDataURL(file);
}

const upload = () => {
  isUploadDone.value = true;

  console.log("finally", base64Files.value);
};
</script>
<template>
  <div
    class="flex flex-col w-auto border border-gray-400 shadow rounded-md pb-3"
  >
    <div class="flex gap-x-10 border-b-2 py-1 px-2 bg-gray-100">
      <input
        ref="fileInput"
        type="file"
        class="hidden"
        :class="[props.inputClass]"
        :accept="props.accept"
        @change="onFileSelect"
        :disabled="disabled"
        :multiple="fileLimit > 1"
      />
      <button
        class="flex-1 rounded-md px-6 py-1 bg-gray-200 disabled:bg-red-500 cursor-pointer disabled:cursor-not-allowed"
        @click="open"
        :disabled="chooseDisabled"
      >
        Select
      </button>
      <button
        @click="upload"
        class="flex-1 rounded-md px-6 py-1 bg-gray-200 cursor-pointer disabled:bg-red-500 disabled:cursor-not-allowed"
        :disabled="uploadDisabled"
      >
        {{ !!init.length || modelValue ? "Update" : "Upload" }}
      </button>
      <button
        @click="clear"
        class="flex-1 rounded-md px-6 py-1 bg-gray-200 cursor-pointer disabled:bg-red-500 disabled:cursor-not-allowed"
        :disabled="cancelDisabled"
      >
        Clear
      </button>
    </div>
    <div class="flex flex-col gap-y-5 w-full mt-3 mb-5 px-2">
      <div
        class="bg-red-100 border-l-[6px] border-red-500 rounded-l-md text-teal-900 px-4 py-3 shadow-md relative"
        role="alert"
        v-for="msg of messages"
        :key="msg"
      >
        <div class="flex">
          <div class="py-1 pr-3">
            <Icon
              name="ion:warning-outline"
              width="35"
              height="35"
              color="#ef4444"
            />
          </div>
          <div>
            <p class="font-bold">{{ msg?.split(",")[1] }}</p>
            <p class="text-sm">
              {{ msg?.split(",")[0] }}
            </p>
          </div>
        </div>
        <button
          @click="onMessageClose"
          class="hover:text-red-500 duration-200 absolute top-2 right-2"
        >
          <Icon name="carbon:close-outline" width="25" height="25" />
        </button>
      </div>
    </div>
    <div class="flex flex-col gap-5 items-start px-2">
      <div
        v-for="(file, index) of files"
        :key="file.name + file.type + file.size"
        class="flex items-center border rounded-md w-full gap-3 px-1 py-3"
      >
        <div class="overflow-hidden w-[150px] rounded-md">
          <img
            role="presentation"
            :alt="file.name"
            :src="file.objectURL"
            class="object-contain"
          />
        </div>
        <div class="w-full flex flex-col justify-start items-start">
          <span class="">{{ file.name }}</span>
          <div class="flex gap-x-3">
            <span class="">{{ formatSize(file.size) }}</span>
            <!-- <span
              v-if="!isUploadDone"
              class="bg-orange-400 py-1 px-3 rounded-full text-white text-xs font-bold"
              >Pending</span
            >
            <span
              v-if="isUploadDone"
              class="bg-green-500 py-1 px-3 rounded-full text-white text-xs font-bold"
              >Uploaded</span
            > -->
          </div>
        </div>
        <button @click="remove(index)">
          <Icon name="carbon:close-outline" width="25" height="25" />
        </button>
      </div>
    </div>
  </div>
</template>
