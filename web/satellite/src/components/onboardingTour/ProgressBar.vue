// Copyright (C) 2020 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <div class="progress-bar-container">
        <div class="progress-bar-container__progress-area">
            <div class="progress-bar-container__progress-area__circle">
                <CheckedImage/>
            </div>
            <div class="progress-bar-container__progress-area__bar"/>
            <div class="progress-bar-container__progress-area__circle">
                <CheckedImage/>
            </div>
            <div class="progress-bar-container__progress-area__bar"/>
            <div class="progress-bar-container__progress-area__circle">
                <CheckedImage/>
            </div>
        </div>
        <div class="progress-bar-container__titles-area">
            <span class="progress-bar-container__titles-area__title">Name Your Project</span>
            <span class="progress-bar-container__titles-area__title api-key-title">Create an API Key</span>
            <span class="progress-bar-container__titles-area__title">Upload Data</span>
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

import CheckedImage from '@/../static/images/common/checked.svg';

@Component({
    components: {
        CheckedImage,
    },
})

export default class ProgressBar extends Vue {}
</script>

<style scoped lang="scss">
    .progress-bar-container {
        width: 100%;

        &__progress-area {
            width: calc(100% - 420px);
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 25px 210px 6px 210px;

            &__circle {
                display: flex;
                justify-content: center;
                align-items: center;
                min-width: 20px;
                height: 20px;
                background-color: #c5cbdb;
                border-radius: 10px;
            }

            &__bar {
                width: 100%;
                height: 4px;
                background-color: #c5cbdb;
            }
        }

        &__titles-area {
            display: flex;
            align-items: center;
            justify-content: space-between;
            padding: 0 188px 0 178px;

            &__title {
                font-family: 'font_regular', sans-serif;
                font-size: 10px;
                line-height: 15px;
                color: rgba(0, 0, 0, 0.4);
            }
        }
    }

    .api-key-title {
        padding: 0 15px 0 0;
    }

    @media screen and (max-width: 800px) {

        .progress-bar-container {

            &__progress-area {
                width: calc(100% - 300px);
                padding: 25px 150px 6px 150px;
            }

            &__titles-area {
                padding: 0 128px 0 118px;
            }
        }
    }
</style>