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
const rawFiles = ref([]);
const messages = ref([]);
const dataUrls = ref([]);
const isIcon = ref(true);
const previewSrc = ref(props.init);
const selectedType = ref([]);
const focused = ref(false);

const selectedTypeIcon = computed(() => {
  if (selectedType.value.length) {
    if (
      selectedType.value.includes("image/*") ||
      selectedType.value.includes("image/png") ||
      selectedType.value.includes("image/jpeg") ||
      selectedType.value.includes("image/webp")
    ) {
      isIcon.value = false;
      return previewSrc.value;
    } else if (selectedType.value.includes("application/pdf")) {
      isIcon.value = true;
      return "/images/pdfIcon.png";
    } else if (
      selectedType.value.includes(".csv") ||
      selectedType.value.includes("text/csv") ||
      selectedType.value.includes("application/vnd.ms-excel") ||
      selectedType.value.includes(
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
      )
    ) {
      isIcon.value = true;
      return "/images/xlsIcon.png";
    } else if (selectedType.value.includes("application/msword")) {
      isIcon.value = true;
      return "/images/docxIcon.png";
    }
  } else {
    isIcon.value = true;
    return "/images/document_icon.png";
  }
});

onMounted(() => {
  let i = 0;
  const urlToObject = async (imageUrl) => {
    fetch(imageUrl)
      .then((response) => response.blob())
      .then((blob) => {
        // Create a File object from the Blob
        const file = new File([blob], "image.jpg", { type: "image/png" });
        file.objectURL = window.URL.createObjectURL(file);
        files.value.push(file);
      });
  };
  while (i < previewSrc.value?.length) {
    console.log(previewSrc.value);
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
  console.log("files", files.value, "na", _files);

  for (let file of _files) {
    if (!isFileSelected(file)) {
      if (validate(file)) {
        if (isImage(file)) {
          file.objectURL = window.URL.createObjectURL(file);
        }

        files.value.push(file);
        console.log("original event", { originalEvent: event });
        console.log("working...", files.value);
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
    dm = 3,
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
  emit("clear");
}

function clearInputElement() {
  fileInput.value = "";
}

function remove(index) {
  console.log(index);
  files.value.splice(index, 1);
  //   emits("update:modelValue", a);
}
</script>
<template>
  <div class="flex flex-col max-w-[20rem]">
    <div class="flex gap-x-10">
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
        class="px-6 py-1 bg-gray-200 disabled:bg-red-500 cursor-pointer disabled:cursor-not-allowed"
        @click="open"
        :disabled="chooseDisabled"
      >
        Select
      </button>
      <button
        class="px-6 py-1 bg-gray-200 cursor-pointer disabled:bg-red-500 disabled:cursor-not-allowed"
        :disabled="uploadDisabled"
      >
        Upload
      </button>
      <button
        @click="clear"
        class="px-6 py-1 bg-gray-200 cursor-pointer disabled:bg-red-500 disabled:cursor-not-allowed"
        :disabled="cancelDisabled"
      >
        Clear
      </button>
    </div>
    <div class="flex flex-col">
      <div
        class="bg-red-100 border-t-4 border-red-500 rounded-b text-teal-900 px-4 py-3 shadow-md flex items-start gap-x-10"
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
        <button @click="onMessageClose" class="hover:text-red-500 duration-200">
          <Icon name="carbon:close-outline" width="25" height="25" />
        </button>
      </div>
    </div>
    <div class="flex flex-col gap-x-5 items-center">
      <div
        v-for="(file, index) of files"
        :key="file.name + file.type + file.size"
      >
        <div class="overflow-hidden w-[10rem] rounded-md">
          <img
            role="presentation"
            :alt="file.name"
            :src="file.objectURL"
            class="object-contain"
          />
        </div>
        <div>
          <span class="">{{ formatSize(file.size) }}</span>
          <!-- <FileUploadBadge :value="badgeValue" :class="cx('badge')" :severity="badgeSeverity" :unstyled="unstyled" :pt="ptm('badge')" /> -->
        </div>
        <div>
          <button @click="remove(index)">X</button>
        </div>
      </div>
    </div>
  </div>
</template>
