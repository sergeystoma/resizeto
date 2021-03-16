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
            <span
                v-if="hasEnoughParameters"
                class="upload__copy"
            >
                <strong>Pick</strong> or <strong>Drag</strong> a source image file.
            </span>

            <span
                v-else
                class="upload__copy upload__copy--bad"
            >
                Need at least width and height parameters.
            </span>

            <input
                ref="file"
                type="file"
                class="upload__input"
                accept="image/*"
                title=""
                @dragenter="dragEnter($event)"
                @dragleave="dragLeave($event)"
                @drop="drop($event)"
                @dragover="dragOver($event)"
                @change="picked($event)"
            >
        </div>
    </div>
</template>

<script>
    export default {
        name: 'App',
        data() {
            return {
                draggingOver: false,
                parameters: [],
                hasEnoughParameters: false,
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
            },

            getParameter(name) {
                return this.parameters.find((x) => x[0] === name);
            },

            hasParameter(name) {
                return this.getParameter(name) != null;
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
                    this.handleImage([...this.$refs.fileImage.files]);

                    this.$refs.file.value = null;
                }
            },

            handleImage() {
            },
        },
    };
</script>

<style lang="scss">
    @import url('https://fonts.googleapis.com/css2?family=Lato:wght@300&display=swap');

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

    #app {
        font-family: Lato, Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;

        color: #2c3e50;
    }

    .upload {
        @include fullsize();

        display: flex;

        justify-content: center;
        align-items: center;

        &:hover {
            .upload__copy {
                &:after {
                    opacity: 1;
                }
            }
        }
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
</style>
