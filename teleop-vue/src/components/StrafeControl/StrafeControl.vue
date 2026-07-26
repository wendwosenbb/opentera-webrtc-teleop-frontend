<template>
  <canvas
    ref="canvas"
    v-bind:width="width"
    v-bind:height="height"
    v-on:touchstart="onTouchStart"
    v-on:touchmove="onTouchMove"
    v-on:touchend="onTouchEnd"
    v-on:touchcancel="onTouchEnd"
    v-on:mousedown="onMouseDown"
    v-on:mouseup="onMouseUp"
    v-on:mousemove="onMouseMove"
    v-on:mouseout="onMouseOut"
  />
</template>

<script>
import { roundRect } from "../Joystick/canvasUtils";

// Single-axis (horizontal only) spring-back control, used to strafe robots
// that can move laterally. Mirrors Joystick.vue's interaction model, but the
// knob only ever moves along x and the background fills the whole (wide,
// short) canvas as a pill instead of an inset square.
export default {
  name: "strafe-control",
  data() {
    return {
      x: 0,
      loopIntervalId: null,
      canvas: null,
      context: null,
      isMouseDown: false,
      activeTouchID: null,
    };
  },
  props: {
    width: {
      type: Number,
      required: true,
    },
    height: {
      type: Number,
      required: true,
    },
    absoluteMaxY: {
      // Maximum lateral velocity in m/s
      type: Number,
      required: true,
    },
    publishingRate: {
      type: Number,
      required: false,
      default: 10, // Hz
    },
  },
  emits: ["strafePositionChange"],
  methods: {
    init() {
      this.initVariables();
      this.drawFrame();
    },
    initVariables() {
      if (!this.isMouseDown) {
        this.x = this.getCenterX();
      }
    },
    emitLoop() {
      this.loopIntervalId = setInterval(
        function() {
          this.emitStrafePosition();
          if (!this.isMouseDown) {
            clearInterval(this.loopIntervalId);
          }
        }.bind(this),
        1000 / this.publishingRate
      );
    },
    onMouseDown(event) {
      if (event.button === 0) {
        this.updateStrafePositionFromMouseEvent(event);
        this.isMouseDown = true;
        this.emitLoop();
      }
    },
    onMouseUp(event) {
      if (event.button === 0) {
        this.onRelease();
      }
    },
    onRelease() {
      this.x = this.getCenterX();
      this.isMouseDown = false;
      this.emitStrafePosition();
      this.drawFrame();
    },
    onMouseMove(event) {
      if (this.isMouseDown) {
        this.updateStrafePositionFromMouseEvent(event);
      }
    },
    onMouseOut() {
      this.x = this.getCenterX();
      this.isMouseDown = false;
      this.emitStrafePosition();
      this.drawFrame();
    },
    onTouchMove(event) {
      this.updateStrafePositionFromMouseEvent(event.touches[0]); // Only use the first touch
    },
    onTouchStart(event) {
      event.preventDefault(); // Prevents scrolling when touching the control
      this.isMouseDown = true;
      this.activeTouchID = event.touches[0].identifier;
      this.updateStrafePositionFromMouseEvent(event.touches[0]); // Only use the first touch
      this.emitLoop();
    },
    onTouchEnd(event) {
      // Make sure the interaction is only stopped if the touchend event was triggered
      // by the finger that was controlling this control and not another.
      for (const changedTouch of event.changedTouches) {
        if (changedTouch.identifier === this.activeTouchID) {
          this.onRelease();
        }
      }
    },
    updateStrafePositionFromMouseEvent(event) {
      const rect = this.canvas.getBoundingClientRect();
      const centerX = this.getCenterX();
      const dragRange = this.getDragRange();

      this.x = event.clientX - rect.left;

      const deltaX = this.x - centerX;
      if (Math.abs(deltaX) > dragRange) {
        this.x = centerX + Math.sign(deltaX) * dragRange;
      }

      this.emitStrafePosition();
      this.drawFrame();
    },
    drawFrame() {
      this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);

      this.drawBackground();
      this.drawKnob();
    },
    drawKnob() {
      if (this.isMouseDown) {
        this.context.fillStyle = "rgba(0, 0, 0, 0.75)";
      } else {
        this.context.fillStyle = "#000000";
      }

      this.context.beginPath();
      this.context.arc(
        this.x,
        this.getCenterY(),
        this.getKnobRadius(),
        0,
        2 * Math.PI
      );
      this.context.fill();
    },
    drawBackground() {
      const centerX = this.getCenterX();
      const centerY = this.getCenterY();
      const dragRange = this.getDragRange();

      //Draw the background pill, filling the whole canvas
      this.context.fillStyle = "#87CEEB";
      roundRect(this.context, 0, 0, this.canvas.width, this.canvas.height, centerY);
      this.context.fill();

      const pointOffset = this.canvas.height / 8;
      const halfPointOffset = pointOffset / 2;

      //draw center cross
      this.context.lineWidth = 2;
      this.context.strokeStyle = "#4682B4";
      this.context.beginPath();
      this.context.moveTo(centerX, centerY - pointOffset);
      this.context.lineTo(centerX, centerY + pointOffset);
      this.context.stroke();
      this.context.beginPath();
      this.context.moveTo(centerX - pointOffset, centerY);
      this.context.lineTo(centerX + pointOffset, centerY);
      this.context.stroke();

      this.context.fillStyle = "#4682B4";

      //draw the left triangle
      const leftTriangleStartX = centerX - dragRange * 0.85;

      this.context.beginPath();
      this.context.moveTo(leftTriangleStartX, centerY);
      this.context.lineTo(
        leftTriangleStartX + pointOffset,
        centerY - halfPointOffset
      );
      this.context.lineTo(
        leftTriangleStartX + pointOffset,
        centerY + halfPointOffset
      );
      this.context.fill();

      //draw the right triangle
      const rightTriangleStartX = centerX + dragRange * 0.85;

      this.context.beginPath();
      this.context.moveTo(rightTriangleStartX, centerY);
      this.context.lineTo(
        rightTriangleStartX - pointOffset,
        centerY - halfPointOffset
      );
      this.context.lineTo(
        rightTriangleStartX - pointOffset,
        centerY + halfPointOffset
      );
      this.context.fill();
    },
    emitStrafePosition() {
      // Emit strafe position as a velocity command, y corresponding to the
      // robot's lateral axis.
      const event = {
        y:
          -((this.x - this.getCenterX()) * this.absoluteMaxY) /
          this.getDragRange(),
      };
      this.$emit("strafePositionChange", event);
    },
    getCenterX() {
      return this.canvas.width / 2;
    },
    getCenterY() {
      return this.canvas.height / 2;
    },
    getKnobRadius() {
      return this.canvas.height / 4;
    },
    getDragRange() {
      return this.canvas.width / 2 - this.getKnobRadius() - 3;
    },
  },
  mounted() {
    this.canvas = this.$refs.canvas;
    this.context = this.canvas.getContext("2d");
    this.init();
  },
  unmounted() {
    clearInterval(this.loopIntervalId);
  },
};
</script>
