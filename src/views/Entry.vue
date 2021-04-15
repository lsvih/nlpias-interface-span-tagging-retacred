<template>
    <div class="interface-panel">
        <div class="text-panel">
            <div class="relation" style="font-weight: bold">Relation: <span style="border: 1px solid #aaa;padding: 5px">{{
                    text['relation']
                }}</span>
            </div>
            <div class="text">
                <span v-for="(token, token_idx) in text['token']" :key="token_idx" style="position: relative"
                      v-bind:data-id="token_idx"
                      :class="[entity_type(token_idx), trigger_clazz(token_idx)]">{{ replace(token) }} <span
                    class="sup" :class="entity_type(token_idx)"
                    v-if="token_idx === text.subj_start || token_idx === text.obj_start">{{
                        token_idx === text.subj_start ? 'SUBJ' : 'OBJ'
                    }}:<b>{{ entity_type(token_idx) }}</b>
                    </span>
                </span>
            </div>
        </div>
        <div class="label-panel">
            <button @click="handleLabel(label)" class="label" v-for="(label, index) in labels"
                    :style="{backgroundColor: colors[index]}"
                    v-bind:key="index">
                {{ label }}
                <span v-if="index === labels.indexOf(predict)" class="auto"></span>
                <span class="shortcut" v-if="shortcuts[index] !== ''">{{ shortcuts[index] }}</span>
            </button>
            <button class="label" @click="deleteLabel">
                Delete<span class="shortcut"><backspace-icon style="width:21px;height: 21px;"/></span>
            </button>
        </div>
        <div class="config-btn">
            <document-icon v-if="!!information.project && (!!information.project.document)"
                           @click.native="openDocument"/>
            <setting-icon @click.native="show_config = true"/>
        </div>
        <div class="indicator">{{ idx + 1 }} / {{ total }}</div>
        <a-progress :percent="( idx + 1 ) / total * 100" :show-info="false" class="progress" :stroke-width="14"
                    stroke-color="blue" stroke-linecap="square"/>
        <config-panel :visible="show_config" :labels="labels" :shortcuts="shortcuts" @close="show_config = false"
                      @save="changeHotkey"/>
    </div>
</template>

<script>

import {schemeSet2, schemeSet3} from 'd3-scale-chromatic'
import DocumentIcon from '../components/icons/document'
import SettingIcon from '../components/icons/setting'
import BackspaceIcon from '../components/icons/backspace'
import ConfigPanel from '../components/ConfigPanel'
import commonMethods from '../common'
import {throttle} from '@/common'

/*
Events who end with [Annotation] is the event for annotaion panel, would be destroy while closing the annotation interface.

schema
emit:
getData(task_id, idx) 向 Main 索取 task_id 任务的第 idx 条数据
labelData(task_id, idx, label) 通知 Main 标注成功 task_id 任务的第 idx 条数据，结果为 label

on:
sentData(text, ?predict) 推送回 getData 请求的数据与预测

 */

export default {
    name: 'Entry',
    components: {DocumentIcon, SettingIcon, BackspaceIcon, ConfigPanel},
    data() {
        return {
            title: 'Interface',
            information: {},
            total: null,
            task_id: null,
            idx: 0,
            colors: [...schemeSet2, ...schemeSet3],
            shortcuts: [],
            labels: [],
            span: [],
            predict: [],
            text: '',
            show_config: false
        }
    },
    created() {
        console.log('Remote Interface Created.')
        this.registerEvents()
        this.initStyle()
    },
    methods: {
        is_sub(idx) {
            return this.text.subj_start <= idx && idx <= this.text.subj_end
        },
        is_obj(idx) {
            return this.text.obj_start <= idx && idx <= this.text.obj_end
        },
        trigger_clazz(idx) {
            if (this.span.length !== 2) return ''
            if (this.span[0] <= idx && this.span[1] >= idx)
                return 'trigger'
            return ''
        },
        entity_type(idx) {
            if (this.is_sub(idx)) return this.text['subj_type']
            if (this.is_obj(idx)) return this.text['obj_type']
            return ''
        },
        initStyle() {
            let sheet = new CSSStyleSheet()
            const entity_type = ['STATE_OR_PROVINCE', 'CITY', 'CRIMINAL_CHARGE', 'TITLE', 'NUMBER', 'DATE', 'PERSON', 'DURATION', 'CAUSE_OF_DEATH', 'COUNTRY', 'URL', 'LOCATION', 'NATIONALITY', 'IDEOLOGY', 'ORGANIZATION', 'RELIGION']
            for (let i = 0; i < entity_type.length; i++) {
                sheet.insertRule(`.${entity_type[i]}{ background-color: ${this.colors[i + 4]}}`, i)
            }
            sheet.insertRule(`.text-panel .trigger{background-color:${this.colors[0]}!important; padding: 5px;}`)
            document.adoptedStyleSheets = [...document.adoptedStyleSheets, sheet]
        },
        ...commonMethods,
        registerEvents() {
            this.$eventBus.on('Destroy[Annotation]', () => {
                this.destroyAnnotator()
            })
            this.$eventBus.on('PressKey[Delete][Annotation]', () => {
                this.deleteLabel()
            })
            this.$eventBus.on('Mount[Annotation]', params => {
                this.initAnnotator(params)
            })
            this.$eventBus.on('sentData[Annotation]', data => {
                let text, label, predict
                [text, label, predict] = data
                this.text = JSON.parse(text)
                this.predict = predict || []
                this.span = label || []
            })
        },
        nextData: throttle(function () {
            this.$eventBus.emit('labelData[Annotation]', [this.task_id, this.idx, this.span])
            if (this.idx === this.information.task.size - 1) {
                this.$message.info('It\'s already the last data.')
                return
            }
            this.idx += 1
            this.getData(this.idx)
            window.getSelection().empty()
        }, 1000),
        handleLabel(label) {
            if (label === 'Previous') { // get previous data
                if (this.idx === 0) { // prevent event while on first data
                    this.$message.info('It\'s already the first data.')
                    return
                }
                this.idx -= 1
                this.getData(this.idx)
            } else if (label === 'Next') {
                this.nextData()
            } else {
                let selector = window.getSelection().getRangeAt(0)
                let start = Number(selector.startContainer.parentElement.dataset.id)
                let end = Number(selector.endContainer.parentElement.dataset.id)
                console.log([start, end])
                this.span = [start, end]
                this.$eventBus.emit('labelData[Annotation]', [this.task_id, this.idx, this.span])
                window.getSelection().empty()
            }
        },
        deleteLabel() {
            this.span = []
        },
        replace(token) {
            switch (token) {
                case '-LRB-':
                    return '('
                case '-RRB-':
                    return ')'
                case '-LSB-':
                    return '['
                case '-RSB-':
                    return ']'
                case '-LCB-':
                    return '{'
                case '-RCB-':
                    return '}'
                default:
                    return token
            }
        }
    }
}
</script>

<style scoped>
.interface-panel {
    width: 100%;
    height: 100%;
    z-index: 2;
    background-color: #fff;
    position: relative;
}

.text-panel {
    border-radius: 20px;
    border: 1px solid #ccc;
    padding: 20px;
    width: 80vw;
    max-height: 60vh;
    overflow-y: auto;
    margin: 10vh 10vw;
    font-size: 22px;
}

.label-panel {
    position: fixed;
    bottom: 5vh;
    margin: 0 10vw;
}

.label {
    position: relative;
    background-color: #ddd;
    height: 60px;
    font-size: 25px;
    padding: 0 25px;
    border-radius: 10px;
    border: none;
    margin-right: 70px;
    cursor: pointer;
}

.label:hover {
    filter: brightness(70%);
}

.shortcut {
    display: inline-block;
    height: 25px;
    width: 25px;
    line-height: 25px;
    vertical-align: middle;
    font-size: 15px;
    color: #000;
    font-weight: bold;
    padding: 0;
    border: 2px solid #000;
    border-radius: 5px;
    opacity: 0.2;
}

.config-btn {
    position: fixed;
    right: 20px;
    bottom: 5vh;
    padding-bottom: 10px;
}

.auto {
    display: inline-block;
    position: absolute;
    height: 10px;
    width: 10px;
    background-color: yellow;
    border-radius: 5px;
    left: 5px;
    top: 5px;
}

.progress {
    position: fixed;
    bottom: 0;
    left: 0;
}

.text {
    line-height: 60px;
    margin: 30px 0;
}

.sup {
    position: absolute;
    line-height: 30px;
    font-size: 15px;
    padding: 0 10px;
    border-radius: 5px 5px 0 0;
    user-select: none;
    top: -30px;
    left: 0;
}

.indicator {
    position: fixed;
    top: 60px;
    right: 10px;
    z-index: 10;
    user-select: none;
    color: #ddd;
    font-size: 30px;
    transition-duration: .5s;
}

.indicator:hover {
    color: #000
}
</style>

<style>
.interface-container .progress .ant-progress-inner {
    border-radius: 0 !important;
    vertical-align: bottom !important;
}

.interface-container button.tagged {
    border: none;
}
</style>