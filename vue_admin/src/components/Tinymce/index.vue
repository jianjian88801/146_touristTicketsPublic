<template>
    <div :class="{fullscreen:fullscreen}" class="tinymce-container" :style="{width:containerWidth}">
        <textarea :id="tinymceId" class="tinymce-textarea" />
        <div class="editor-custom-btn-container">
            <editorImage color="#1890ff" class="editor-upload-btn" @successCBK="imageSuccessCBK" />
        </div>
    </div>
</template>

<script>
    /**
     * docs:
     * https://panjiachen.github.io/vue-element-admin-site/feature/component/rich-editor.html#tinymce
     */
    import editorImage from "./components/EditorImage";
    import plugins from "./plugins";
    import toolbar from "./toolbar";
    import load from "./dynamicLoadScript";

    // why use this cdn, detail see https://github.com/PanJiaChen/tinymce-all-in-one
    const tinymceCDN = "/static/tinymce/tinymce.min.js";

    export default {
        name: "jt-tinymce",
        components: { editorImage },
        props: {
            id: {
                type: String,
                default: function () {
                    return "vue-tinymce-" + +new Date() + ((Math.random() * 1000).toFixed(0) + "");
                },
            },
            value: {
                type: String,
                default: "",
            },
            toolbar: {
                type: Array,
                required: false,
                default() {
                    return [];
                },
            },
            menubar: {
                type: String,
                default: "file edit insert view format table",
            },
            height: {
                type: [Number, String],
                required: false,
                default: 360,
            },
            width: {
                type: [Number, String],
                required: false,
                default: "auto",
            },
            uploadImage: {
                type: Function,
                required: true,
            },
            pickerUploadImage: {
                type: Function,
                default: function () {
                    return function (file, success, failure) {};
                },
            },
        },
        data() {
            return {
                hasChange: false,
                hasInit: false,
                tinymceId: this.id,
                fullscreen: false,
                languageTypeList: {
                    en: "en",
                    zh: "zh_CN",
                    es: "es_MX",
                    ja: "ja",
                },
            };
        },
        computed: {
            containerWidth() {
                const width = this.width;
                if (/^[\d]+(\.[\d]+)?$/.test(width)) {
                    // matches `100`, `'100'`
                    return `${width}px`;
                }
                return width;
            },
        },
        watch: {
            value(val) {
                if (!this.hasChange && this.hasInit) {
                    this.$nextTick(() => window.tinymce.get(this.tinymceId).setContent(val || ""));
                }
            },
        },
        mounted() {
            this.init();
        },
        activated() {
            if (window.tinymce) {
                this.initTinymce();
            }
        },
        deactivated() {
            this.destroyTinymce();
        },
        destroyed() {
            this.destroyTinymce();
        },
        methods: {
            init() {
                // dynamic load tinymce from cdn
                load(tinymceCDN, (err) => {
                    if (err) {
                        this.$message.error(err.message);
                        return;
                    }
                    this.initTinymce();
                });
            },
            initTinymce() {
                const _this = this;
                window.tinymce.init({
                    selector: `#${this.tinymceId}`,
                    language: this.languageTypeList["zh"],
                    height: this.height,
                    body_class: "panel-body ",
                    object_resizing: false,
                    paste_data_images: true,
                    toolbar: this.toolbar.length > 0 ? this.toolbar : toolbar,
                    menubar: this.menubar,
                    plugins: plugins,
                    end_container_on_empty_block: true,
                    powerpaste_word_import: "clean",
                    code_dialog_height: 450,
                    code_dialog_width: 1000,
                    branding: false,
                    automatic_uploads: true,
                    advlist_bullet_styles: "square",
                    advlist_number_styles: "default",
                    imagetools_cors_hosts: ["www.tinymce.com", "codepen.io"],
                    default_link_target: "_blank",
                    link_title: false,
                    nonbreaking_force_tab: true, // inserting nonbreaking space &nbsp; need Nonbreaking Space Plugin
                    init_instance_callback: (editor) => {
                        if (_this.value) {
                            editor.setContent(_this.value);
                        }
                        _this.hasInit = true;
                        editor.on("NodeChange Change KeyUp SetContent", () => {
                            this.hasChange = true;
                            this.$emit("input", editor.getContent());
                        });
                    },
                    setup(editor) {
                        editor.on("FullscreenStateChanged", (e) => {
                            _this.fullscreen = e.state;
                        });
                    },
                    // it will try to keep these URLs intact
                    // https://www.tiny.cloud/docs-3x/reference/configuration/Configuration3x@convert_urls/
                    // https://stackoverflow.com/questions/5196205/disable-tinymce-absolute-to-relative-url-conversions
                    convert_urls: false,
                    images_upload_handler(blobInfo, success, failure) {
                        _this.uploadImage(blobInfo.blob(), success, failure);
                    },

                    //拖放方式 上传图片
                    file_picker_callback: function (callback, value, meta) {
                        var input = document.createElement("input");
                        input.setAttribute("type", "file");

                        input.onchange = function () {
                            var file = this.files[0];
                            if (file) {
                                _this.uploadImage(file, callback, function () {});
                            }

                            /*_this.pickerUploadImage(file, callback , function () {

                            })*/
                        };
                        input.click();
                    },
                    paste_upload_image: function (editor, file, success, error) {
                        _this.uploadImage(file, success, error);
                    },
                });
            },
            destroyTinymce() {
                const tinymce = window.tinymce.get(this.tinymceId);
                if (this.fullscreen) {
                    tinymce.execCommand("mceFullScreen");
                }
                if (tinymce) {
                    tinymce.destroy();
                }
            },
            setContent(value) {
                window.tinymce.get(this.tinymceId).setContent(value);
            },
            getContent() {
                window.tinymce.get(this.tinymceId).getContent();
            },
            imageSuccessCBK(arr) {
                arr.forEach((v) => window.tinymce.get(this.tinymceId).insertContent(`<img class="wscnph" src="${v.url}" >`));
            },
        },
    };
</script>

<style lang="scss" scoped>
    .tinymce-container {
        position: relative;
        line-height: normal;
    }

    .tinymce-container {
        ::v-deep {
            .mce-fullscreen {
                z-index: 10000;
            }
        }
    }

    .tinymce-textarea {
        visibility: hidden;
        z-index: -1;
    }

    .editor-custom-btn-container {
        position: absolute;
        right: 4px;
        top: 4px;
        /*z-index: 2005;*/
    }

    .fullscreen .editor-custom-btn-container {
        z-index: 10000;
        position: fixed;
    }

    .editor-upload-btn {
        display: inline-block;
    }
</style>
