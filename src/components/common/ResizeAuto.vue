// ResizeAuto.vue
<script>
export default {
  name: "ResizeAuto",
  
  mounted() {
    this.$nextTick(() => {
      this.$el.style.height = "auto";
      this.$el.style.height = `${this.$el.scrollHeight}px`;
      this.$el.style.resize = "none"
      this.$el.style.overflow = "hidden"
    });
  },
  updated() {
    this.$nextTick(() => {
      this.$el.style.height = "auto";
      this.$el.style.height = `${this.$el.scrollHeight}px`;
      this.$el.style.resize = "none"
      this.$el.style.overflow = "hidden"
    });
  },
  render() {
    return this.$scopedSlots.default({
      //resize: this.resize,
    });
  },
  
};
</script>