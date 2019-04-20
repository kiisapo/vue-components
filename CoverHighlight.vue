<template>
    <div class="cover-highlight">
        <div class="backdrop">
            <div ref="highlights" class="highlights" v-html="dealText" />
        </div>
        <textarea ref="textarea" v-model="inputText" @input="change" @scroll="scroll" :placeholder="placeholder" :spellcheck="false" />
        <div class="word-count">{{currLength}}/{{maxLength}}</div>
    </div>
</template>

<script>
    export default {
        data (){
            return {
                // 输入标志
                inputFlag: true,

                // //处理特殊字符的中转器
                transfer: document.createElement('div'),

                // 原始输入的文本
                inputText: this.value,

                // 处理后并加上违禁词的文本
                dealText: '',

                // 当前字数
                currLength: 0
            }
        },

        props: {
            // 双向绑定
            value: {
                type: String,
                default: ''
            },

            // 最大输入限制
            maxLength: {
                type: Number,
                default: 10000
            },

            // 文本域是否可以编辑
            canEdit: {
                type: Boolean,
                default: true
            },

            // 绝对不允许的禁词
            taboo: {
                type: Array,
                default: []
            },

            // 敏感词
            sensitive: {
                type: Array,
                default: []
            },

            // 输入占位
            placeholder: {
                type: String,
                default: ''
            }
        },

        mounted (){
            if(this.inputText){
                this.change();
            };

            // 添加compositionstart、compositionend事件
            // 目前vue中对这两个事件处理不太友好，需要手动绑定
            let textarea = this.$refs.textarea;
            if(textarea.addEventListener){
                textarea.addEventListener('oncompositionstart', this.compositionstart, false);
                textarea.addEventListener('oncompositionend', this.compositionend, false);
            };
        },

        watch: {
            value (val){
                this.inputText = val;
                this.change();
            }
        },

        methods: {

            /**
             * 处理输入
             */
            change (){
                if(this.inputFlag){
                    // 先清除格式
                    let text = this.clearHTML(this.inputText);

                    // 限制最大输入
                    if(text.length >= this.maxLength){
                        this.inputText = text.substring(0, this.maxLength);
                        return false;
                    };

                    // 返回干净的文本
                    this.$emit('input', this.inputText);

                    // 匹配敏感词
                    this.dealText = this.matchWord(this.inputText);

                    // 统计字数
                    this.currLength = this.inputText.length;
                };
            },

            /**
             * 清除复制、输入进来的html格式
             * 
             * 通过获取标签内的纯文本，干掉一些特殊标签，如<template>；
             * 为什么要这么处理，而不用正则替换？
             * 因为outerText获取到的是纯文本，没有html标签，而正则替换需要些一堆匹配，还会有考虑不到的地方；
             */
            clearHTML (content){
                this.transfer.innerHTML = content;
                return this.transfer.outerText;
            },

            /**
             * 匹配敏感词
             * 
             * 在干净的文本中加入高亮
             */
            matchWord (content){
                for (let i = 0; i < this.taboo.length; i++) {
                    let word = this.taboo[i];
                    content = content.replace(new RegExp(word,'g'), "<span class='label-taboo'>$&</span>");
                };

                for(let i = 0; i < this.sensitive.length; i++){
                    let word = this.sensitive[i];
                    content = content.replace(new RegExp(word,'g'), "<span class='label-sensitive'>$&</span>");
                };

                content += '<br>';

                return content;
            },

            /**
             * 输入开始时触发
             */
            compositionstart (){
                console.log('compositionstart');
                this.inputFlag = false;
                return false;
            },

            /**
             * 选择字/词完成输入时触发
             */
            compositionend (){
                console.log('compositionend');
                this.inputFlag = true;
                return false;
            },

            /**
             * 文本域滚动时设置背景的偏移
             * 这里只设置了Y轴，Y轴同理
             * 
             * 注意：为什么要用translate，而不是设置背景的滚动条
             * 是因为不同的浏览器对滚动条处理不一样，有几率不会叠加到一起
             */
            scroll (e){
                this.$refs.highlights.style.transform = `translateY(-${e.target.scrollTop}px)`;
            }
        },

        /**
         * 销毁实例时移除事件
         */
        beforeDestroy (){
            let textarea = this.$refs.textarea;

            if(textarea.removeEventListener){
                textarea.removeEventListener('oncompositionstart', this.compositionstart);
                textarea.removeEventListener('oncompositionend', this.compositionend);
            };
        }
    }
</script>

<style lang="scss">
    .cover-highlight{
        width: 100%;
        height: 100%;
        display: block;
        -webkit-text-size-adjust: none;
        position: relative;
        border: 1px solid #DCDFE6;
        border-radius: 4px;
        box-sizing: border-box;
        overflow: hidden;

        & > .backdrop, & > textarea{
            width: 100%;
            height: 100%;
            overflow: auto;
            outline: none;
            position: absolute;
            box-sizing: border-box;
            padding: 10px;
            -webkit-appearance: textarea;
            -webkit-rtl-ordering: logical;
            -webkit-writing-mode: horizontal-tb !important;
            flex-direction: column;
            white-space: pre-wrap;
            word-wrap: break-word;
            text-rendering: auto;
            word-spacing: normal;
            text-transform: none;
            text-indent: 0px;
            text-align: start;
            margin: 0em;
            font: 14px/22px 'Open Sans', sans-serif;
            letter-spacing: 1px;
        }

        .backdrop{
            z-index: 1;
            background-color: #fff;
            overflow: auto;
            pointer-events: none;

            .highlights{
                white-space: pre-wrap;
                word-wrap: break-word;
                color: transparent;

                .label-taboo{
                    background: red;
                    border-radius: 3px;
                }

                .label-sensitive{
                    background: orange;
                    border-radius: 3px;
                }
            }
        }

        textarea{
            display: block;
            z-index: 2;
            margin: 0;
            border: none;
            color: #444;
            background-color: transparent;
            overflow: auto;
            resize: none;
        }

        .word-count{
            height: 20px;
            color: #444;
            background: #fff;
            position: absolute;
            bottom: 0px;
            right: 0;
            z-index: 3;
            line-height: 20px;
            padding: 0 5px;
            border-radius: 3px;
        }
    }
</style>
