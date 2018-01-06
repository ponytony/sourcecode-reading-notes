<template>
    <div :class="classes">
        <button :class="arrowClasses" class="left" @click="arrowEvent(-1)">
            <Icon type="chevron-left"></Icon>
        </button>
        <div :class="[prefixCls + '-list']">
            <div :class="[prefixCls + '-track', showCopyTrack ? '' : 'higher']" :style="trackStyles" ref="originTrack">
                <slot></slot>
            </div>
            <div :class="[prefixCls + '-track', showCopyTrack ? 'higher' : '']" :style="copyTrackStyles" ref="copyTrack" v-if="loop">
            </div>
        </div>
        <button :class="arrowClasses" class="right" @click="arrowEvent(1)">
            <Icon type="chevron-right"></Icon>
        </button>
        <ul :class="dotsClasses">
            <template v-for="n in slides.length">
                <li :class="[n - 1 === currentIndex ? prefixCls + '-active' : '']"
                    @click="dotsEvent('click', n - 1)"
                    @mouseover="dotsEvent('hover', n - 1)">
                    <button :class="[radiusDot ? 'radius' : '']"></button>
                </li>
            </template>
        </ul>
    </div>
</template>
<script>
    import Icon from '../icon/icon.vue';
    import { getStyle, oneOf } from '../../utils/assist';
    import { on, off } from '../../utils/dom';
    /*
    oneof: 第一个参数是被检查的参数，参数二是参数列表，用来判断参数一是否在参数二中，返回bool
    getstyle：参数一是节点，参数二是style名，用来检验节点的那个属性值
    on和off是对dom添加事件的配适器
     */

    const prefixCls = 'ivu-carousel';

    export default {
        name: 'Carousel',
        components: { Icon },
        props: {
            arrow: {
                type: String,
                default: 'hover',
                validator (value) {
                    return oneOf(value, ['hover', 'always', 'never']);
                }
            },
            autoplay: {
                type: Boolean,
                default: false
            },
            autoplaySpeed: {
                type: Number,
                default: 2000
            },
            loop: {
                type: Boolean,
                default: false
            },
            easing: {
                type: String,
                default: 'ease'
            },
            dots: {
                type: String,
                default: 'inside',
                validator (value) {
                    return oneOf(value, ['inside', 'outside', 'none']);
                }
            },
            radiusDot: {
                type: Boolean,
                default: false
            },
            trigger: {
                type: String,
                default: 'click',
                validator (value) {
                    return oneOf(value, ['click', 'hover']);
                }
            },
            value: {
                type: Number,
                default: 0
            },
            height: {
                type: [String, Number],
                default: 'auto',
                validator (value) {
                    return value === 'auto' || Object.prototype.toString.call(value) === '[object Number]';
                }
            }
        },
        data () {
            return {
                prefixCls: prefixCls,
                listWidth: 0,
                trackWidth: 0,
                trackOffset: 0,
                trackCopyOffset: 0,
                showCopyTrack: false,
                slides: [],
                slideInstances: [], // 用于放置slide节点
                timer: null,
                ready: false,
                currentIndex: this.value,
                trackIndex: this.value,
                copyTrackIndex: this.value,
                hideTrackPos: -1, // 默认左滑
            };
        },
        computed: {
            classes () {
                return [
                    `${prefixCls}`
                ];
            },
            trackStyles () {
                return {
                    width: `${this.trackWidth}px`,
                    transform: `translate3d(${-this.trackOffset}px, 0px, 0px)`,
                    transition: `transform 500ms ${this.easing}`
                };
            },
            copyTrackStyles () {
                return {
                    width: `${this.trackWidth}px`,
                    transform: `translate3d(${-this.trackCopyOffset}px, 0px, 0px)`,
                    transition: `transform 500ms ${this.easing}`,
                    position: 'absolute',
                    top: 0
                };
            },
            arrowClasses () {
                return [
                    `${prefixCls}-arrow`,
                    `${prefixCls}-arrow-${this.arrow}`
                ];
            },
            dotsClasses () {
                return [
                    `${prefixCls}-dots`,
                    `${prefixCls}-dots-${this.dots}`
                ];
            }
        },
        methods: {
            // 寻找子节点中有componentName的那个节点，其实找的就是carousel-item,虽然也可以找到其他的部件？？？
            findChild (cb) {
                const find = function (child) {
                    const name = child.$options.componentName;

                    // 如果有componentName
                    if (name) {
                        cb(child);
                    } else if (child.$children.length) {
                      // 如果没有componentName但是child还有子节点，就继续向下递归
                        child.$children.forEach((innerChild) => {
                            find(innerChild, cb);// 感觉这儿可以不加第二个参数，因为作用域是可以寻找到最顶层的cb的
                        });
                    }
                };

                // 如果有节点或者找不到子节点
                if (this.slideInstances.length || !this.$children) {
                    this.slideInstances.forEach((child) => {
                        find(child);
                    });
                } else {
                    this.$children.forEach((child) => {
                        find(child);
                    });
                }
            },
            // 怀疑是用来给走马灯增加下方标签的， 但是官方文档没有给出loop这个属性
          // 也只是给子组件使用的一个函数
            initCopyTrackDom () {
                this.$nextTick(() => {
                    this.$refs.copyTrack.innerHTML = this.$refs.originTrack.innerHTML;
                });
            },
          // 用来更新slides属性，如果参数是true， 那么就同步更新slideInstance属性，在更新的同时会给ele加上一个index属性，方便控制
            updateSlides (init) {
                let slides = [];
                let index = 1;
                this.findChild((child) => {
                    slides.push({
                        $el: child.$el
                    });
                    child.index = index++;
                    if (init) {
                        this.slideInstances.push(child);
                    }
                });

                this.slides = slides;
                this.updatePos();
            },
          // 更新走马灯的高度和宽度，更新整个滑轨的宽度，滑轨指的是走马灯的主体
            updatePos () {
                this.findChild((child) => {
                    child.width = this.listWidth;
                    child.height = typeof this.height === 'number' ? `${this.height}px` : this.height;
                });

                this.trackWidth = (this.slides.length || 0) * this.listWidth;
            },
            // use when slot changed
          // 这个是给carousel-item用的，
            slotChange () {
                this.$nextTick(() => {
                    this.slides = [];
                    this.slideInstances = [];

                    this.updateSlides(true, true);
                    this.updatePos();
                    this.updateOffset();
                });
            },
          // 初始化单个页面的宽度，通过getStyle，然后执行updatePos和updateOffset
            handleResize () {
                this.listWidth = parseInt(getStyle(this.$el, 'width'));
                this.updatePos();
                this.updateOffset();
            },
          // 更新track的index
            updateTrackPos (index) {
                if (this.showCopyTrack) {
                    this.trackIndex = index;
                } else {
                    this.copyTrackIndex = index;
                }
            },
          // 与上方的函数正好相反
            updateTrackIndex (index) {
                if (this.showCopyTrack) {
                    this.copyTrackIndex = index;
                } else {
                    this.trackIndex = index;
                }
            },
          /**
           * 与dotevent事件类似，也是通过trackIndex属性来推动页面
           *
           */
            add (offset) {
                // 获取单个轨道的图片数
                let slidesLen = this.slides.length;
                // 如果是无缝滚动，需要初始化双轨道位置
                if (this.loop) {
                    if (offset > 0) {
                        // 初始化左滑轨道位置
                        this.hideTrackPos = -1;
                    } else {
                        // 初始化右滑轨道位置
                        this.hideTrackPos = slidesLen;
                    }
                    this.updateTrackPos(this.hideTrackPos);
                }
                // 获取当前展示图片的索引值
                let index =  this.showCopyTrack ? this.copyTrackIndex : this.trackIndex;
                index += offset;
                while (index < 0) index += slidesLen;
                if (((offset > 0 && index === slidesLen) || (offset < 0 && index === slidesLen - 1)) && this.loop) {
                    // 极限值（左滑：当前索引为总图片张数， 右滑：当前索引为总图片张数 - 1）切换轨道
                    this.showCopyTrack = !this.showCopyTrack;
                    this.trackIndex += offset;
                    this.copyTrackIndex += offset;
                } else {
                    if (!this.loop) index = index % this.slides.length; // 所以loop为false也会循环？
                    this.updateTrackIndex(index);
                }
                this.$emit('input', index === this.slides.length ? 0 : index);
            },
          /**
           * 给按钮用的方法，集合了两个方法
           */
            arrowEvent (offset) {
                this.setAutoplay();
                this.add(offset);
            },
          // 通过更新trackIndex的方法直接跳到点击到的dot的index指定页面上
            dotsEvent (event, n) {
                let curIndex = this.showCopyTrack ? this.copyTrackIndex : this.trackIndex;
                if (event === this.trigger && curIndex !== n) {
                    this.updateTrackIndex(n);
                    this.$emit('input', n);
                    // Reset autoplay timer when trigger be activated
                    this.setAutoplay();
                }
            },
          /**
           * 先清除定时器
           * 再根据autoplay参数决定是否添加一个定时器
           * 定时add（1）
           */
            setAutoplay () {
                window.clearInterval(this.timer);
                if (this.autoplay) {
                    this.timer = window.setInterval(() => {
                        this.add(1);
                    }, this.autoplaySpeed);
                }
            },
          // 不过就是更新走马灯偏移值offset
            updateOffset () {
                this.$nextTick(() => {
                    /* hack: revise copyTrack offset (1px) */
                    let ofs = this.copyTrackIndex > 0 ? -1 : 1;
                    this.trackOffset = this.trackIndex * this.listWidth;
                    this.trackCopyOffset = this.copyTrackIndex * this.listWidth + ofs; // 这里这个hack看不懂
                });
            }
        },
        watch: {
            autoplay () {
                this.setAutoplay();
            },
            autoplaySpeed () {
                this.setAutoplay();
            },
            currentIndex (val, oldVal) {
                this.$emit('on-change', oldVal, val);
            },
            trackIndex () {
                this.updateOffset();
            },
            copyTrackIndex () {
                this.updateOffset();
            },
            height () {
                this.updatePos();
            },
            value (val) { // 这里很奇怪，我明明在外部传入了新的value，还是不会更新着两个index
                this.currentIndex = val;
                this.trackIndex = val;
            }
        },
        mounted () {
            this.updateSlides(true);
            this.handleResize();
            this.setAutoplay();
//            window.addEventListener('resize', this.handleResize, false);
            on(window, 'resize', this.handleResize);
        },
        beforeDestroy () {
          // 卸载插件的时候将event取消掉
//            window.removeEventListener('resize', this.handleResize, false);
            off(window, 'resize', this.handleResize);
        }
    };
</script>
