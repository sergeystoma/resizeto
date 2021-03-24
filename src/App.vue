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

        <div
            class="upload__docs"
        >
            <p>
                Resizeto.com is an image resizing tool aimed to quickly resize images for documentation and user guides where all images should have somewhat standard dimensions and file formats.
            </p>
            <p>
                <strong>How to use:</strong>
            </p>
            <p>
                Create a link describing the desired image parameters such as width, height, padding, etc, and share that link with documentation authors. For example, <a href="https://resizeto.com/w_1000/h_500/f_fit/bg_ffffffff/as_jpg/q_75">https://resizeto.com/w_1000/h_500/f_fit/bg_ffffffff/as_jpg/q_75</a>
            </p>
            <p class="mt">
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
            <p>
                <strong>frame</strong> &mdash; Wrap the image into light (<strong>browserlight</strong>) or dark (<strong>browserdark</strong>) browser mockup window, or <strong>shadow</strong> for a simple drop shadow frame.
            </p>
            <p class="mt">
                All processing happens inside your browser window and no images are uploaded to servers. No tracking of any kind. Source code, bug reports, and requests at <strong><a href="https://github.com/sergeystoma/resizeto">https://github.com/sergeystoma/resizeto</a></strong>
            </p>
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
            ref="framed"
        />

        <canvas
            ref="framedMask"
        />

        <canvas
            ref="target"
        />
    </div>
</template>

<script>
    import { saveAs } from 'file-saver';

    const pica = require('pica')();

    const frames = {
        browserlight: {
            image: '/frames/browserlight.png',
            mask: '/frames/browserlightmask.png',
            patch: {
                left: 120,
                top: 75,
                right: 30,
                bottom: 30,
            },
            pad: {
                left: 15,
                top: 74,
                right: 15,
                bottom: 15,
            },
        },
        browserdark: {
            image: '/frames/browserdark.png',
            mask: '/frames/browserdarkmask.png',
            patch: {
                left: 120,
                top: 75,
                right: 30,
                bottom: 30,
            },
            pad: {
                left: 15,
                top: 74,
                right: 15,
                bottom: 15,
            },
        },
        shadow: {
            image: '/frames/shadow.png',
            mask: '/frames/shadowmask.png',
            patch: {
                left: 30,
                top: 30,
                right: 30,
                bottom: 30,
            },
            pad: {
                left: 15,
                top: 15,
                right: 15,
                bottom: 15,
            },
        },
    };

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
            this.handlePaste();
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

                    const px = this.getParameter('px');
                    const py = this.getParameter('py');

                    let pad = 'no padding';

                    if (px > 0 || py > 0) {
                        pad = `${px}px &times; ${py}px padding`;
                    }

                    this.description = `Will ${fit} ${w}px &times; ${h}px <i>${min}</i> with ${pad} and save as a ${this.describeFormat(as)} file.`;
                }
            },

            getParameter(name) {
                const p = this.parameters.find((x) => x[0] === name);

                const value = p ? p[1] : null;

                if (name === 'w') {
                    return /[0-9]+/.test(value) ? parseInt(value, 10) : null;
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

                if (name === 'frame') {
                    return value;
                }

                return null;
            },

            getFrame() {
                const f = this.getParameter('frame');

                return frames[f];
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

            padZero(v, count) {
                let padded = v.toString();

                while (padded.length < count) {
                    padded += '0';
                }

                return padded;
            },

            handlePaste() {
                document.onpaste = (event) => {
                    const items = (event.clipboardData || event.originalEvent.clipboardData).items;

                    items.forEach((item) => {
                        if (item.kind === 'file') {
                            const blob = item.getAsFile();

                            const now = new Date();
                            const name = `pasted-${now.getFullYear()}-${this.padZero(now.getMonth())}-${this.padZero(now.getDate())}`;

                            this.handleImage([blob], name);
                        }
                    });
                };
            },

            handleImage(files, name) {
                this.done = false;

                if (this.doneTimer) {
                    clearTimeout(this.doneTimer);
                }

                if (files.length > 0) {
                    const file = files[0];

                    const reader = new FileReader();
                    reader.onload = () => {
                        const loadTasks = [];

                        // Load the main image.
                        const image = new Image();
                        loadTasks.push(new Promise((resolve) => {
                            image.onload = () => {
                                resolve();
                            };
                        }));
                        image.src = reader.result;

                        // Load frames.
                        const frame = this.getFrame();

                        if (frame) {
                            frame.imageImage = new Image();
                            loadTasks.push(new Promise((resolve) => {
                                frame.imageImage.onload = () => {
                                    resolve();
                                };
                            }));
                            frame.imageImage.src = frame.image;

                            frame.maskImage = new Image();
                            loadTasks.push(new Promise((resolve) => {
                                frame.maskImage.onload = () => {
                                    resolve();
                                };
                            }));
                            frame.maskImage.src = frame.mask;
                        }

                        Promise.all(loadTasks).then(() => {
                            this.processImage(name ?? file.name, image);
                        });
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

                let availableW = Math.max(0, w - px - px);
                let availableH = Math.max(0, h - py - py);

                let targetW;
                let targetH;
                let outsideW;
                let outsideH;
                let croppedW;
                let croppedH;
                let framedW;
                let framedH;

                const frame = this.getFrame();
                if (frame) {
                    availableW = Math.max(0, availableW - frame.pad.left - frame.pad.right);
                    availableH = Math.max(0, availableH - frame.pad.top - frame.pad.bottom);
                }

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

                    croppedW = targetW;
                    croppedH = targetH;

                    if (frame) {
                        framedW = targetW + frame.pad.left + frame.pad.right;
                        framedH = targetH + frame.pad.top + frame.pad.bottom;
                    } else {
                        framedW = targetW;
                        framedH = targetH;
                    }

                    break;
                }

                case 'cover':
                {
                    if (frame) {
                        outsideW = availableW;
                        outsideH = availableH;

                        framedW = availableW + frame.pad.left + frame.pad.right;
                        framedH = availableH + frame.pad.top + frame.pad.bottom;
                    } else {
                        outsideW = availableW;
                        outsideH = availableH;

                        framedW = outsideW;
                        framedH = outsideH;
                    }

                    croppedW = outsideW;
                    croppedH = outsideH;

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

                    croppedW = targetW;
                    croppedH = targetH;

                    if (frame) {
                        framedW = targetW + frame.pad.left + frame.pad.right;
                        framedH = targetH + frame.pad.top + frame.pad.bottom;

                        outsideH += frame.pad.top + frame.pad.bottom;
                    } else {
                        framedW = targetW;
                        framedH = targetH;
                    }

                    break;
                }
                }

                return {
                    outsideW,
                    outsideH,
                    framedW,
                    framedH,
                    croppedW,
                    croppedH,
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

            drawNine(ctx, w, h, image, o) {
                const iw = image.width;
                const ih = image.height;

                // Left-top.
                ctx.drawImage(image, 0, 0, o.left, o.top, 0, 0, o.left, o.top);
                // Right-top.
                ctx.drawImage(image, iw - o.right, 0, o.right, o.top, w - o.right, 0, o.right, o.top);
                // Left-bottom.
                ctx.drawImage(image, 0, ih - o.bottom, o.left, o.bottom, 0, h - o.bottom, o.left, o.bottom);
                // Right-bottom.
                ctx.drawImage(image, iw - o.right, ih - o.bottom, o.right, o.bottom, w - o.right, h - o.bottom, o.right, o.bottom);

                // Top.
                ctx.drawImage(image, o.left, 0, iw - o.left - o.right, o.top, o.left, 0, w - o.left - o.right, o.top);
                // Bottom.
                ctx.drawImage(image, o.left, ih - o.bottom, iw - o.left - o.right, o.bottom, o.left, h - o.bottom, w - o.left - o.right, o.bottom);
                // Left.
                ctx.drawImage(image, 0, o.top, o.left, ih - o.top - o.bottom, 0, o.top, o.left, h - o.top - o.bottom);
                // Right.
                ctx.drawImage(image, iw - o.right, o.top, o.right, ih - o.top - o.bottom, w - o.right, o.top, o.right, h - o.top - o.bottom);

                // Center.
                ctx.drawImage(image, o.left, o.top, iw - o.left - o.right, ih - o.top - o.bottom, o.left, o.top, w - o.left - o.right, h - o.top - o.bottom);
            },

            cropImage(name, target) {
                // Crop the resized image to desired dimensions.
                this.resizeCanvas(this.$refs.cropped, target.croppedW, target.croppedH);

                const croppedCtx = this.$refs.cropped.getContext('2d');
                croppedCtx.clearRect(0, 0, target.croppedW, target.croppedH);

                croppedCtx.drawImage(
                    this.$refs.sized,
                    Math.round((target.croppedW - target.targetW) / 2),
                    Math.round((target.croppedH - target.targetH) / 2));

                // Add a frame.
                const frame = this.getFrame();
                if (frame) {
                    // Prepare a mask.
                    this.resizeCanvas(this.$refs.framedMask, target.framedW, target.framedH);

                    const maskCtx = this.$refs.framedMask.getContext('2d');
                    maskCtx.clearRect(0, 0, target.framedW, target.framedH);

                    this.drawNine(maskCtx, target.framedW, target.framedH, frame.maskImage, frame.patch);

                    // Draw cropped image.
                    this.resizeCanvas(this.$refs.framed, target.framedW, target.framedH);

                    const framedCtx = this.$refs.framed.getContext('2d');
                    framedCtx.clearRect(0, 0, target.framedW, target.framedH);

                    framedCtx.drawImage(this.$refs.cropped, frame.pad.left, frame.pad.top);

                    // Mask the image.
                    framedCtx.globalCompositeOperation = 'destination-in';
                    framedCtx.drawImage(this.$refs.framedMask, 0, 0);

                    // Add the frame.
                    framedCtx.globalCompositeOperation = 'source-over';
                    this.drawNine(framedCtx, target.framedW, target.framedH, frame.imageImage, frame.patch);
                } else {
                    this.resizeCanvas(this.$refs.framed, target.framedW, target.framedH);

                    const framedCtx = this.$refs.framed.getContext('2d');
                    framedCtx.clearRect(0, 0, target.framedW, target.framedH);

                    framedCtx.drawImage(this.$refs.cropped, 0, 0);
                }

                // Center in the final canvas size.
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

                targetCtx.drawImage(this.$refs.framed, target.px + ((target.outsideW - target.framedW) / 2), target.py + ((target.outsideH - target.framedH) / 2));

                this.saveImage(name);
            },

            saveImage(name) {
                let mimeType = '';
                let extension = '';

                switch (this.getParameter('as')) {
                case 'png':
                    mimeType = 'image/png';
                    extension = '.png';
                    break;
                case 'jpg':
                    mimeType = 'image/jpeg';
                    extension = '.jpg';
                    break;
                }

                const quality = this.getParameter('q');

                let saveName = name;
                let dotIndex = name.lastIndexOf('.');
                if (dotIndex > 0) {
                    saveName = saveName.substr(0, dotIndex);
                }
                saveName += extension;

                pica.toBlob(this.$refs.target, mimeType, quality).then((blob) => {
                    saveAs(blob, saveName);

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

        border: 1px solid #000;
    }

    a {
        color: #05aff2;

        text-decoration: none;
    }

    p {
        font-size: 14px;
    }

    #app {
        max-width: 800px;
        margin: auto;

        font-family: Lato, Helvetica, Arial, sans-serif;
        line-height: 1.6;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;

        color: #2c3e50;
    }

    .upload {
        position: relative;
        display: flex;

        justify-content: flex-start;
        align-items: center;

        flex-direction: column;

        height: 300px;

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

        opacity: 0.5;
    }

    .upload__copy {
        position: relative;

        margin-top: 100px;

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
