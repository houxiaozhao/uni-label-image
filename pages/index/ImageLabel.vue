<template>
  <div class="relative">
    <canvas id="canvas" canvas-id="canvas" style="width: 100%; height: 400px; border-bottom: 1rpx #888 solid" @touchstart="touchstart" @touchmove="touchmove" @touchend="touchend" @touchcancel="touchcancel" disable-scroll @click="clickCanvas"></canvas>
    <div>
      <button @click="mode = 'PAN'">PAN</button>
      <button @click="mode = 'RECT'">RECT</button>
    </div>
    {{ rects }}
  </div>
</template>

<script>
let ctx;
export default {
  name: "ImageLabel",
  props: {
    url: {
      type: String,
      required: true,
    },
    isEdit: {
      type: Boolean,
      default: true,
    },
    initFeatures: {
      type: Array,
      default: () => {
        return [];
      },
    },
  },
  data() {
    return {
      touches: [], // 双指位置
      touch: {
        x: 0,
        y: 1,
      }, // 单指位置

      imgX: 0, // 图片左上角相对于 canvas 左上角的X轴位置
      imgY: 0, // 图片左上角相对于 canvas 左上角的 Y轴位置
      isMove: false, // 是否移动
      imgScale: 1, // 图片缩放比例, 实际长度到 canvas 长度需要乘，canvas 到实际需要除
      MINIMUM_SCALE: 0.2, // 最小缩放
      MAX_SCALE: 5, // 最大缩放
      canvas: {
        //canvas 的长宽
        height: 0,
        width: 0,
      },
      img: {
        // 图片信息
        path: "",
        width: 0,
        height: 0,
      },
      mode: "PAN",
      rects: [],
      rect: {
        start: {
          x: 0,
          y: 0,
        },
        // 矩形相对于 image 的位置
        x: 0,
        y: 0,
        w: 0,
        h: 0,
      },
    };
  },
  async mounted() {
    ctx = uni.createCanvasContext("canvas");
    this.canvas = await this.getCanvasWH("#canvas");
    await this.loadImage(this.url);
    //  把图片缩放到刚好可以全部看到的大小
    if (this.img.width / this.img.height > this.canvas.width / this.canvas.height) {
      this.imgScale = this.canvas.width / this.img.width;
      this.imgX = 0;
      this.imgY = (this.canvas.height - this.img.height * this.imgScale) / 2;
    } else {
      this.imgScale = this.canvas.height / this.img.height;
      this.imgX = (this.canvas.width - this.img.width * this.imgScale) / 2;
      this.imgY = 0;
    }
    this.draw();
  },
  methods: {
    clickCanvas(e) {
      // console.log("e: " + JSON.stringify(e.detail));
      const point = e.detail;
      //canvas坐标转image坐标
      // console.log((e.detail.x - this.imgX) / this.imgScale);
      // console.log((e.detail.y - this.imgY) / this.imgScale);
      const width = 10;
      if (this.mode !== "PAN") {
        //其他模式可以编辑
        //检查是否点击到了矩形或多边形上
        // 因为矩形多边形的边框像素是不变的，所以把矩形多边形的边框的坐标转换为canvas上的坐标来判断
        const rect = this.rects.find(rect => {
          console.log(rect);
          const x = rect.x * this.imgScale + this.imgX;
          const y = rect.y * this.imgScale + this.imgY;
          const w = rect.w * this.imgScale;
          const h = rect.h * this.imgScale;
          // 判断点是否在width宽的矩形内的依据是在外框内，在内框外
          const x1 = x - width / 2;
          const y1 = y - width / 2;
          const w1 = w + width;
          const h1 = h + width;
          const x2 = x + width / 2;
          const y2 = y + width / 2;
          const w2 = w - width;
          const h2 = h - width;
          const inOuterFrame = point.x >= x1 && point.y >= y1 && point.x <= x1 + w1 && point.y <= y1 + h1;
          const inInnerFrame = point.x >= x2 && point.y >= y2 && point.x <= x2 + w2 && point.y <= y2 + h2;
          return inOuterFrame && !inInnerFrame;
        });
        console.log("点到了rect", rect);
        this.rects.forEach(e => (e.selected = false));
        if (rect) rect.selected = true;
        this.draw();
      }
    },
    touchstart(e) {
      console.log("开始触摸", e);
      const { touches } = e;
      if (this.mode === "PAN") {
        this.isMove = true;
        // 判断是否为多手指
        if (Object.keys(touches).length < 2) {
          this.touch = touches[0];
        } else {
          this.touches = touches;
        }
      }
      if (this.mode === "RECT") {
        this.rect.start.x = (touches[0].x - this.imgX) / this.imgScale; //相对于图片的位置，缩放后
        this.rect.start.y = (touches[0].y - this.imgY) / this.imgScale; //相对于图片的位置，缩放后
      }
    },
    touchmove(e) {
      console.log("移动 " + JSON.stringify(e.touches));
      if (this.mode === "PAN") {
        if (!this.isMove || !e.touches) return false;
        // 如果是单指
        if (Object.keys(e.touches).length === 1) {
          const now = e.touches[0]; // 当前位置
          const x = now.x - this.touch.x; //和上一次位置对比，移动的距离
          const y = now.y - this.touch.y;
          this.imgX += x; //把移动的距离添加到图片相对 canvas 的位置
          this.imgY += y;
          this.touch = now; // 更新最新位置
        } else {
          const start = this.touches; //移动前两个手指位置
          const end = e.touches; //移动前两个手指位置
          const startDistance = this.getDistance(start[0], start[1]); //移动前两个手指距离
          const endDistance = this.getDistance(end[0], end[1]); //移动后两个手指距离
          // 根据距离计算缩放比例

          const imgXStart = this.imgX; //移动前图片位置
          const imgYStart = this.imgY;
          const imgScaleStart = this.imgScale; //移动前图片缩放

          const pointsInImage = [
            {
              //两个手指相对图片的位置
              x: (start[0].x - imgXStart) / imgScaleStart,
              y: (start[0].y - imgYStart) / imgScaleStart,
            },
            {
              x: (start[1].x - imgXStart) / imgScaleStart,
              y: (start[1].y - imgYStart) / imgScaleStart,
            },
          ];
          let imgScaleEnd = (imgScaleStart * endDistance) / startDistance; //移动后图片缩放
          if (imgScaleEnd >= this.MAX_SCALE) imgScaleEnd = this.MAX_SCALE; //防止过大
          if (imgScaleEnd <= this.MINIMUM_SCALE) imgScaleEnd = this.MINIMUM_SCALE; // 防止过小

          const imgXEnd = end[0].x - pointsInImage[0].x * imgScaleEnd; //移动后图片位置
          const imgYEnd = end[0].y - pointsInImage[0].y * imgScaleEnd;

          this.imgScale = imgScaleEnd;
          this.imgX = imgXEnd;
          this.imgY = imgYEnd;

          this.touches = e.touches;
        }
      }
      if (this.mode === "RECT") {
        const x = (e.touches[0].x - this.imgX) / this.imgScale;
        const y = (e.touches[0].y - this.imgY) / this.imgScale;

        this.rect.x = Math.min(x, this.rect.start.x);
        this.rect.y = Math.min(y, this.rect.start.y);
        this.rect.w = Math.abs(x - this.rect.start.x);
        this.rect.h = Math.abs(y - this.rect.start.y);
      }
      this.draw();
    },
    touchend(e) {
      console.log("触摸结束", e);
      if (this.mode === "PAN") {
        this.isMove = false;
      }
      if (this.mode === "RECT") {
        if (this.rect.w !== 0 && this.rect.h !== 0)
          this.rects.push(
            Object.assign(this.rect, {
              id: Date.now(),
            })
          );
        this.rect = {
          start: {
            x: 0,
            y: 0,
          },
          x: 0,
          y: 0,
          h: 0,
          w: 0,
        };
      }
      this.draw();
    },
    touchcancel(e) {
      if (this.mode === "PAN") {
        this.isMove = false;
      }
      if (this.mode === "RECT") {
        if (this.rect.w !== 0 && this.rect.h !== 0)
          this.rects.push(
            Object.assign(this.rect, {
              id: Date.now(),
            })
          );
        this.rect = {
          start: {
            x: 0,
            y: 0,
          },
          x: 0,
          y: 0,
          h: 0,
          w: 0,
        };
      }
      this.draw();
    },
    getCanvasWH(id) {
      return new Promise((resolve, reject) => {
        const query = uni.createSelectorQuery().in(this);
        query
          .select(id)
          .boundingClientRect(data => {
            resolve({
              height: data.height,
              width: data.width,
            });
          })
          .exec();
      });
    },
    draw() {
      ctx.drawImage(this.img.path, 0, 0, this.img.width, this.img.height, this.imgX, this.imgY, this.img.width * this.imgScale, this.img.height * this.imgScale);
      ctx.setStrokeStyle("#00f");
      ctx.setLineWidth(1);
      if (this.rect.h !== 0 && this.rect.w !== 0) {
        ctx.strokeRect(this.rect.x * this.imgScale + this.imgX, this.rect.y * this.imgScale + this.imgY, this.rect.w * this.imgScale, this.rect.h * this.imgScale);
      }
      this.rects.forEach(rect => {
        if (rect.selected) {
          ctx.setStrokeStyle("red");
          ctx.setLineWidth(3);
        } else {
          ctx.setStrokeStyle("#00f");
          ctx.setLineWidth(1);
        }
        ctx.strokeRect(rect.x * this.imgScale + this.imgX, rect.y * this.imgScale + this.imgY, rect.w * this.imgScale, rect.h * this.imgScale);
      });
      ctx.draw();
    },
    getDistance(p1, p2) {
      const x = p2.x - p1.x;
      const y = p2.y - p1.y;
      return Math.sqrt(x * x + y * y);
    },
    async loadImage(url) {
      const info = await uni.getImageInfo({
        src: url,
      });
      this.img = info;
      console.log("info: " + JSON.stringify(info));
    },
  },
};
</script>

<style></style>
