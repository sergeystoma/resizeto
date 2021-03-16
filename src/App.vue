<template>
    <div id="app">
        <div
            :class="[
                'upload',
                {
                    'upload--over': draggingOver,
                }
            ]"
        >
            <div
                v-if="hasEnoughParameters"
                class="upload__copy"
            >
                <strong>Pick</strong> or <strong>Drag</strong> a source image file.
            </div>

            <div
                :class="[
                    'upload__done',
                    {
                        'upload__done--done': done,
                    }
                ]"
            >
                <strong>Done</strong>, downloading your image...
            </div>


            <div
                v-if="hasEnoughParameters"
                class="upload__description"
            >
                <span v-html="description" />
            </div>

            <div
                v-if="!hasEnoughParameters"
                class="upload__copy upload__copy--bad"
            >
                Need at least width and height parameters.
            </div>

            <div
                class="upload__docs"
            >
                <p>
                    Resizeto.com is a tool aimed to quickly resize images for documentation and user guide where all images should have somewhat standard dimensions and file formats.
                </p>
                <p>
                    <strong>w</strong> &mdash; All generated images will have this width.
                </p>
                <p>
                    <strong>h</strong> &mdash; Height guideline, the exact result dimensions will depend on the fitting mode.
                </p>
                <p>
                    <strong>f</strong> &mdash; Fitting mode. <strong>fit</strong> to create <i>w &times; h</i> image and fit the source image completely inside. <strong>cover</strong> to create <i>w &times; h</i> image and size the source image to cover the area; some cropping may occur. <strong>extend</strong> to create an image with <i>w</i> width and the dynamic height of at least <i>h</i> to match the source image aspect ratio.
                </p>
                <p>
                    <strong>px</strong> &mdash; Horizontal padding.
                </p>
                <p>
                    <strong>py</strong> &mdash; Vertical padding.
                </p>
                <p>
                    <strong>as</strong> &mdash; <i>png</i> or <i>jpg</i>.
                </p>
                <p>
                    <strong>q</strong> &mdash; JPEG image quality.
                </p>
                <p>
                    <strong>bg</strong> &mdash; Resulting image background in RRGGBBAA hex format, i.e. FFFFFFFF is fully opaque white, FF00007F is semi-transparent red, etc.
                </p>
                <p class="mt">
                    For example, <a href="https://resizeto.com/w_1000/h_500/f_fit/bg_ffffffff/as_jpg/q_75">https://resizeto.com/w_1000/h_500/f_fit/bg_ffffffff/as_jpg/q_75</a>
                </p>
                <p class="mt">
                    All processing happens inside your browser window and no images are uploaded to servers. No tracking of any kind. Source code, bug reports, and requests at <strong><a href="https://github.com/sergeystoma/resizeto">https://github.com/sergeystoma/resizeto</a></strong>
                </p>
            </div>

            <input
                ref="file"
                type="file"
                class="upload__input"
                accept="image/png,image/jpeg,image/gif"
                title=""
                @dragenter="dragEnter($event)"
                @dragleave="dragLeave($event)"
                @drop="drop($event)"
                @dragover="dragOver($event)"
                @change="picked($event)"
            >
        </div>

        <canvas
            ref="source"
        />

        <canvas
            ref="sized"
        />

        <canvas
            ref="cropped"
        />

        <canvas
            ref="target"
        />
    </div>
</template>

<script>
    import { saveAs } from 'file-saver';

    const pica = require('pica')();

    export default {
        name: 'App',
        data() {
            return {
                draggingOver: false,
                parameters: [],
                hasEnoughParameters: false,
                done: false,
            };
        },
        mounted() {
            this.parseParameters();
        },
        methods: {
            parseParameters() {
                this.parameters = window.location.pathname
                    .split('/')
                    .filter((x) => x && x.length > 0)
                    .map((x) => x.split('_'))
                    .filter((x) => x.length === 2);

                this.hasEnoughParameters = this.hasParameter('w') && this.hasParameter('h');

                if (this.hasEnoughParameters) {
                    const w = this.getParameter('w');
                    const h = this.getParameter('h');
                    const as = this.getParameter('as');

                    let fit = '';
                    let min = '';
                    switch (this.getParameter('f')) {
                    case 'fit':
                        fit = 'fit to';
                        break;

                    case 'cover':
                        fit = 'cover';
                        break;

                    case 'extend':
                        fit = 'extend to';
                        min = 'minimum';
                        break;
                    }

                    this.description = `Will ${fit} ${w}px &times; ${h}px <i>${min}</i> and save as a ${this.describeFormat(as)} file.`;
                }
            },

            getParameter(name) {
                const p = this.parameters.find((x) => x[0] === name);

                const value = p ? p[1] : null;

                if (name === 'w') {
                    return /[0-9]+/.test(p[1]) ? parseInt(value, 10) : null;
                }

                if (name === 'h') {
                    return /[0-9]+/.test(value) ? parseInt(value, 10) : null;
                }

                if (name === 'px') {
                    return /[0-9]+/.test(value) ? parseInt(value, 10) : 0;
                }

                if (name === 'py') {
                    return /[0-9]+/.test(value) ? parseInt(value, 10) : 0;
                }

                if (name === 'f') {
                    return value === 'fit' || value === 'cover' || value === 'extend' ? value : 'fit';
                }

                if (name === 'bg') {
                    return /[0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f]/.test((value || '').toLowerCase()) ? value.toLowerCase() : null;
                }

                if (name === 'q') {
                    return /[0-9]+/.test(value) ? (parseInt(value, 10) / 100) : 1;
                }

                if (name === 'as') {
                    return value === 'png' || value === 'jpg' ? value : 'jpg';
                }

                return null;
            },

            hasParameter(name) {
                return this.getParameter(name) != null;
            },

            describeFormat(as) {
                if (as === 'png') {
                    return 'PNG';
                }

                if (as === 'jpg') {
                    return 'JPEG';
                }

                return '';
            },

            dragEnter() {
                this.draggingOver = true;
            },

            dragOver(e) {
                e.preventDefault();
            },

            dragLeaveImage() {
                this.draggingOver = false;
            },

            drop(e) {
                e.preventDefault();

                this.draggingOver = false;

                if (e.dataTransfer && e.dataTransfer.files) {
                    this.handleImage([...e.dataTransfer.files]);
                }
            },

            picked() {
                if (this.$refs.file.files.length > 0) {
                    this.handleImage([...this.$refs.file.files]);

                    this.$refs.file.value = null;
                }
            },

            handleImage(files) {
                this.done = false;

                if (this.doneTimer) {
                    clearTimeout(this.doneTimer);
                }

                if (files.length > 0) {
                    const file = files[0];

                    const reader = new FileReader();
                    reader.onload = () => {
                        const image = new Image();
                        image.onload = () => {
                            this.processImage(file.name, image);
                        };
                        image.src = reader.result;
                    };
                    reader.readAsDataURL(file);
                }
            },

            resizeCanvas(canvas, w, h) {
                canvas.width = w;
                canvas.height = h;
                canvas.style.width = `${w}px`;
                canvas.style.height = `${h}px`;
            },

            loadSource(image) {
                this.resizeCanvas(this.$refs.source, image.width, image.height);

                const sourceCtx = this.$refs.source.getContext('2d');
                sourceCtx.clearRect(0, 0, image.width, image.height);
                sourceCtx.drawImage(image, 0, 0);
            },

            calculateTargetDimensions(image) {
                const w = this.getParameter('w');
                const h = this.getParameter('h');
                const px = this.getParameter('px');
                const py = this.getParameter('py');

                const availableW = Math.max(0, w - px - px);
                const availableH = Math.max(0, h - py - py);

                let targetW;
                let targetH;
                let outsideW;
                let outsideH;

                switch (this.getParameter('f')) {
                case 'fit':
                {
                    outsideW = availableW;
                    outsideH = availableH;

                    targetW = image.width;
                    targetH = image.height;

                    if (targetW > availableW) {
                        const f = availableW / targetW;

                        targetW *= f;
                        targetH *= f;
                    }

                    if (targetH > availableH) {
                        const f = availableH / targetH;

                        targetW *= f;
                        targetH *= f;
                    }

                    break;
                }

                case 'cover':
                {
                    outsideW = availableW;
                    outsideH = availableH;

                    targetW = image.width;
                    targetH = image.height;

                    if (targetW < availableW) {
                        const f = availableW / targetW;

                        targetW *= f;
                        targetH *= f;
                    }

                    if (targetH < availableH) {
                        const f = availableH / targetH;

                        targetW *= f;
                        targetH *= f;
                    }

                    break;
                }

                case 'extend':
                {
                    outsideW = availableW;

                    targetW = image.width;
                    targetH = image.height;

                    if (targetW > availableW) {
                        const f = availableW / targetW;

                        targetW *= f;
                        targetH *= f;
                    }

                    outsideH = targetH;

                    if (outsideH < availableH) {
                        outsideH = availableH;
                    }

                    break;
                }
                }

                return {
                    outsideW,
                    outsideH,
                    targetW,
                    targetH,
                    px,
                    py,
                };
            },

            resizeSource(name, image, target) {
                this.resizeCanvas(this.$refs.sized, target.targetW, target.targetH);

                const sizedCtx = this.$refs.sized.getContext('2d');
                sizedCtx.clearRect(0, 0, target.targetW, target.targetH);

                pica.resize(this.$refs.source, this.$refs.sized, {
                    quality: 3,
                    alpha: true,
                }).then(() => {
                    this.cropImage(name, target);
                });
            },

            cropImage(name, target) {
                this.resizeCanvas(this.$refs.cropped, target.outsideW, target.outsideH);

                const croppedCtx = this.$refs.cropped.getContext('2d');
                croppedCtx.clearRect(0, 0, target.outsideW, target.outsideH);

                croppedCtx.drawImage(
                    this.$refs.sized,
                    Math.round((target.outsideW - target.targetW) / 2),
                    Math.round((target.outsideH - target.targetH) / 2));

                const finalW = target.outsideW + target.px + target.px;
                const finalH = target.outsideH + target.py + target.py;

                this.resizeCanvas(this.$refs.target, finalW, finalH);

                const targetCtx = this.$refs.target.getContext('2d');

                let bg = this.getParameter('bg');

                if (bg == null && this.getParameter('as') === 'jpg') {
                    bg = 'ffffffff';
                }

                if (bg) {
                    const r = parseInt(bg.substr(0, 2), 16);
                    const g = parseInt(bg.substr(2, 2), 16);
                    const b = parseInt(bg.substr(4, 2), 16);
                    const a = parseInt(bg.substr(6, 2), 16);

                    targetCtx.fillStyle = `rgba(${r}, ${g}, ${b}, ${a / 255})`;

                    targetCtx.fillRect(0, 0, finalW, finalH);
                } else {
                    targetCtx.clearRect(0, 0, finalW, finalH);
                }

                targetCtx.drawImage(this.$refs.cropped, target.px, target.py);

                this.saveImage(name);
            },

            saveImage(name) {
                let mimeType = '';

                switch (this.getParameter('as')) {
                case 'png':
                    mimeType = 'image/png';
                    break;
                case 'jpg':
                    mimeType = 'image/jpeg';
                    break;
                }

                const quality = this.getParameter('q');

                pica.toBlob(this.$refs.target, mimeType, quality).then((blob) => {
                    saveAs(blob, name);

                    this.done = true;

                    this.doneTimer = setTimeout(() => {
                        this.done = false;
                    }, 3000);
                });
            },

            processImage(name, image) {
                this.loadSource(image);

                const target = this.calculateTargetDimensions(image);

                this.resizeSource(name, image, target);
            }
        },
    };
</script>

<style lang="scss">
    @import url('https://fonts.googleapis.com/css2?family=Lato:wght@400;900&display=block');

    @mixin fullsize() {
        position: absolute;

        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
    }

    body, html {
        @include fullsize();

        margin: 0;
        padding: 0;
    }

    strong {
        font-weight: 900;
    }

    canvas {
        display: none;
    }

    a {
        color: #000;

        text-decoration: none;
    }

    #app {
        font-family: Lato, Helvetica, Arial, sans-serif;
        line-height: 1.6;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;

        color: #2c3e50;
    }

    .upload {
        @include fullsize();

        display: flex;

        justify-content: center;
        align-items: center;

        flex-direction: column;

        &:hover {
            .upload__copy {
                &:after {
                    opacity: 1;
                }
            }
        }
    }

    .upload__description {
        margin-top: 25px;

        font-size: 16px;
    }

    .upload__copy {
        position: relative;

        font-size: 20px;

        &:after {
            content: '';

            position: absolute;

            left: 0;
            bottom: -5px;
            height: 2px;
            width: 100%;

            background: #000;

            opacity: 0;
            transition: opacity 0.2s ease-out;
        }

        &.upload__copy--bad {
            color: #d00;

            &:after {
                display: none;
            }
        }
    }

    .upload__input {
        @include fullsize();

        cursor: pointer;

        outline: none;

        opacity: 0;
    }

    .upload__docs {
        position: relative;

        z-index: 1;

        width: 800px;
        max-width: calc(100vw - 40px);

        margin-top: 50px;
    }

    .upload__done {
        margin-top: 20px;

        color: #056cf2;

        font-size: 20px;

        opacity: 0;

        transition: opacity 0.25s ease-out;

        &.upload__done--done {
            opacity: 1;
        }
    }

    .mt {
        margin-top: 20px;
    }
</style>
