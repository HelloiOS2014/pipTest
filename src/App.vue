<template>
  <div class="container" @contextmenu.prevent="rightEmptyClick($event)" id="container">
      <div v-for="(columnItem, index) in chartList" class="chart_container" :key="index">
        <div class="list_header" :style="{ width: columnItem.roadOption.width + 'px' }" draggable="true"
          @drop="dropEvent($event, index)" @dragstart="drag($event, index)" @dragover.prevent
          @contextmenu.prevent="rightHeaderClick($event, columnItem)">
          <div class="list_header_title list_header_line list_header_bottom_line">
            <label>{{ columnItem.roadOption.headerTitle }}</label>
          </div>
          <div v-if="columnItem.roadOption.isEmpty">
            <div class="list_header_title list_header_empty_line">
              <label>{{ columnItem.roadOption.emptyDesc }}</label>
            </div>
          </div>
          <div v-else>
            <div style="width: 100%; height: 60px; overflow: auto">
              <div v-for="item in columnItem.curveDesc" v-bind:key="item">
                <div class="list_header_title list_header_line list_header_bottom_line">
                  <label>{{ item.curveName }}</label>
                </div>
                <div class="list_header_rate list_header_title list_header_line">
                  <label v-for="rateData in item.rate" v-bind:key="rateData">{{
                    rateData
                    }}</label>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="listChart"
          :style="{ width: columnItem.roadOption.width + 'px', height: columnItem.roadOption.max * columnItem.roadOption.box * columnItem.roadOption.scale + 10 + 'px' }"
          :id="columnItem.chartId" @contextmenu.prevent="rightClick($event)" />
      </div>
    <div class="mask_container" v-if="showMask" v-on:click="maskClick" @contextmenu.prevent="maskClick" />
  </div>
</template>

<script>
import echarts from "echarts";
import { markRaw } from "vue";
import axios from "axios";

export default {
  data() {
    return {
      domList: [],
      chartObjs: {},
      columns: [],
      chartList: [],
      clickDomId: "",
      minPosition: [-1, -1],
      maxPosition: [-1, -1],
      allowDomId: "",
      drawing: false,
      mouseDown: false,
      showMask: false,
    };
  },
  mounted() {
    const that = this;
    window.onresize = () =>
      (() => {
        that.chartResize();
      })();
    window.startDrugCount = 0;
    window.addChart = this.addChart;
    window.updataChart = this.updataChart;
    window.deleteChart = this.deleteChart;
    window.changeMask = this.changeMask;
    // this.initTest();
    this.readJson();
  },
  methods: {
    chartResize() {
      this.$nextTick(() => {
        for (const key in this.chartObjs) {
          let chart = this.chartObjs[key];
          chart.resize();
        }
      });
    },
    registerEvent() {
      let that = this;
      if (this.allowDomId == "") {
        return;
      }
      const element = document.getElementById(this.allowDomId);
      element.onmousedown = (e) => {
        if (that.allowDomId == "") {
          return;
        }
        let chatrItemIdx = e.path.findIndex((ele) => {
          return ele.id != undefined && ele.id != "";
        });
        if (chatrItemIdx == -1) {
          return;
        }
        let chatrItem = e.path[chatrItemIdx];
        let domId = chatrItem.id;
        if (domId == undefined) {
          return;
        }
        if (domId != that.allowDomId) {
          return;
        }

        let chart = that.chartObjs[domId];

        if (chart == undefined) {
          return;
        }

        const pointInPixel = [e.offsetX, e.offsetY];

        const selectPostion = chart.convertFromPixel(
          { seriesIndex: 0 },
          pointInPixel
        );
        const selectY = selectPostion[1];
        if (that.minPosition[1] == -1 || that.maxPosition[1] == -1) {
          return;
        }
        if (selectY >= that.minPosition[1] && selectY <= that.maxPosition[1]) {
          that.clickDomId = domId;
          that.mouseDown = true;
        }
      };
      element.onmousemove = (e) => {
        if (!that.mouseDown) {
          return;
        }
        if (that.allowDomId == "") {
          return;
        }
        let chatrItemIdx = e.path.findIndex((ele) => {
          return ele.id != undefined && ele.id != "";
        });
        if (chatrItemIdx == -1) {
          that.drawing = false;
          return;
        }
        let chatrItem = e.path[chatrItemIdx];
        let domId = chatrItem.id;
        if (domId == "" || that.clickDomId == "") {
          that.drawing = false;
          return;
        }
        if (domId == that.clickDomId) {
          let chart = that.chartObjs[domId];

          if (chart == undefined) {
            that.drawing = false;
            return;
          }

          that.drawing = true;

          let opt = chart.getOption();
          const pointInPixel = [e.offsetX, e.offsetY];

          const selectPostion = chart.convertFromPixel(
            { seriesIndex: 0 },
            pointInPixel
          );

          opt.series[opt.series.length - 1].data.push(selectPostion);
          chart.setOption(opt);
        }
      };

      element.onmouseup = (e) => {
        if (that.allowDomId == "") {
          return;
        }
        let chart = that.chartObjs[that.allowDomId];

        if (chart == undefined) {
          that.endDraw();
          return;
        }

        let chartOpt = chart.getOption();
        if (chartOpt == undefined) {
          that.endDraw();
          return;
        }
        if (that.maxPosition[1] == -1) {
          return;
        }
        chartOpt.series[chartOpt.series.length - 1].data.push(that.maxPosition);
        chart.setOption(chartOpt);
        that.endDraw();
      };
    },
    addLister() {
      let that = this;
      for (let index = 0; index < this.domList.length; index++) {
        const element = this.domList[index];
        let chart = that.chartObjs[element];
        if (chart == undefined) {
          continue;
        }
        // chart.getZr().on("mousewheel", (res) => {
        //   if (res.wheelDelta > 0) {
        //     that.scrollChart(true);
        //   } else {
        //     that.scrollChart(false);
        //   }
        // });
        let chartData = this.chartList[index];
        if (chartData.roadOption.isMerge) {
          continue;
        }
        chart.getZr().on("click", (params) => {
          const pointInPixel = [params.offsetX, params.offsetY];
          if (params.offsetX > -5 && params.offsetX < 5) {
            that.clickPixel(pointInPixel);
          } else {
            that.mergePixel(chart, pointInPixel, index);
          }
        });
      }
    },
    readJson() {
      let that = this;
      axios.get("/static/data/pipeline.json").then((res) => {
        that.updataChart(res.data);
      });
    },
    joinSort(originalList, joinList) {
      var resultList = new Array(originalList.length + joinList.length);
      var i = 0,
        j = 0;
      while (i < originalList.length || j < joinList.length) {
        resultList[i + j] =
          i == originalList.length
            ? joinList[j++]
            : j == joinList.length
              ? originalList[i++]
              : originalList[i][1] < joinList[j][1]
                ? originalList[i++]
                : joinList[j++];
      }
      return resultList;
    },
    scrollChart(isUp) {
      // return
      for (let index = 0; index < this.domList.length; index++) {
        const domId = this.domList[index];
        let chart = this.chartObjs[domId];
        let data = this.chartList[index];
        if (chart == undefined) {
          continue;
        }

        if (data == undefined) { continue; }

        let option = chart.getOption();
        if (option == undefined) {
          continue;
        }
        let dataZoom = option.dataZoom[0];
        const addValue = isUp ? -1 : 1;
        data.option.dataZoom[0].start += addValue;
        data.option.dataZoom[0].end += addValue;
        this.chartList[index] = data;
        chart.dispatchAction({
          type: "dataZoom",
          start: dataZoom.start + addValue,
          end: dataZoom.end + addValue,
        });
      }
    },
    drag(event, index) {
      window.startDrugCount = index;
    },
    dropEvent(event, newIndex) {
      let oldIndex = window.startDrugCount;
      if (oldIndex == newIndex) {
        return;
      }
      let tempChartList = this.chartList;
      let oldData = tempChartList[oldIndex];
      let newData = tempChartList[newIndex];
      if (oldData == undefined || newData == undefined) {
        return;
      }
      let oldDomId = oldData.chartId;
      let newDomId = newData.chartId;
      if (oldDomId == undefined || newDomId == undefined) {
        return;
      }

      let isChange = false;
      if (event.clientY != undefined) {
        isChange = event.clientY <= 31;
      }

      if (
        oldData.roadOption.isEmpty ||
        newData.roadOption.isEmpty ||
        isChange
      ) {
        this.chartObjs[oldDomId] = undefined;
        this.chartObjs[newDomId] = undefined;
        tempChartList[oldIndex] = newData;
        tempChartList[newIndex] = oldData;
        this.chartList = tempChartList;
        this.domList[oldIndex] = newDomId;
        this.domList[newIndex] = oldDomId;
        let that = this;
        this.$nextTick(() => {
          that.changeChart(oldIndex, newIndex, true);
        });
      } else {
        let oldOpt = oldData.option;
        let newOpt = newData.option;
        if (oldOpt == undefined || newOpt == undefined) {
          return;
        }
        newOpt.series = newOpt.series.concat(oldOpt.series);
        oldOpt.series = [];
        newData.option = newOpt;
        oldData.option = oldOpt;

        newData.curveDesc = newData.curveDesc.concat(oldData.curveDesc);

        oldData.curveDesc = [];

        this.chartObjs[oldDomId] = undefined;
        this.chartObjs[newDomId] = undefined;

        this.chartList[oldIndex] = oldData;
        this.chartList[newIndex] = newData;

        let that = this;
        this.sendMessageToNative("drop", {
          oldIdx: oldIndex,
          newIdx: newIndex,
        });
        that.$nextTick(() => {
          that.changeChart(oldIndex, newIndex, false);
        });
      }
    },
    endDraw() {
      this.mouseDown = false;
      this.clickDomId = "";
      this.allowDomId = "";
      this.minPosition = [-1, -1];
      this.maxPosition = [-1, -1];
    },
    rightEmptyClick(event) {
      if (this.domList.length > 0) {
        return;
      }
      this.sendMessageToNative("rightClick", null);
    },
    rightClick(event) {
      if (this.domList.length == 0) {
        return;
      }
      if (event.path.length < 3) {
        return;
      }
      let data = event.path[2];
      if (data == undefined) {
        return;
      }
      if (data.id != undefined) {
        this.sendMessageToNative("rightClick", {
          chartId: data.id,
        });
      }
    },
    rightHeaderClick(event, columnItem) {
      if (columnItem == undefined) {
        return;
      }
      this.sendMessageToNative("rightClick", {
        chartId: columnItem.chartId,
      });
    },
    maskClick() {
      this.sendMessageToNative("maskClick", {
        clickMask: true,
      });
    },
    changeMask(status) {
      this.showMask = status;
    },

    mergePixel(chart, pointInPixel, index) {
      let chartData = this.chartList[index];
      if (chartData == undefined) {
        return;
      }
      let checkResult = chartData.roadOption.merged;
      if (checkResult != undefined && checkResult) {
        return;
      }
      this.chartList[index].roadOption.merged = true;

      let op = chartData.option;
      if (op == undefined) {
        return;
      }
      let mergeIndex = this.chartList.findIndex((currentValue) => {
        if (currentValue.roadOption.isMerge == undefined) {
          return false;
        }
        return currentValue.roadOption.isMerge;
      });
      if (mergeIndex == -1) {
        return;
      }

      let mergeData = this.chartList[mergeIndex];

      if (mergeData == undefined) {
        return;
      }

      let mergeOp = mergeData.option;
      if (mergeOp == undefined) {
        return;
      }

      let mergedomId = this.domList[mergeIndex];
      if (mergedomId == undefined) {
        return;
      }

      let mergeChart = this.chartObjs[mergedomId];
      if (mergeChart == undefined) {
        return;
      }

      let pointInGrid = chart.convertFromPixel(
        { seriesIndex: 0 },
        pointInPixel
      );
      let yIndex = pointInGrid[1];

      let markLine = op.series[0].markLine;

      if (markLine == undefined) {
        return;
      }

      let markLineList = markLine.data;

      if (markLineList == undefined) {
        return;
      }

      let startPoint = undefined;
      for (let idx = 0; idx < markLineList.length; idx++) {
        const element = markLineList[idx].yAxis;
        if (startPoint == undefined) {
          if (element < yIndex) {
            startPoint = element;
          }
        } else {
          if (element < yIndex && element > startPoint) {
            startPoint = element;
          }
        }
      }

      let endPoint = undefined;
      for (let idx = 0; idx < markLineList.length; idx++) {
        const element = markLineList[idx].yAxis;
        if (endPoint == undefined) {
          if (element > yIndex) {
            endPoint = element;
          }
        } else {
          if (element > yIndex && element < endPoint) {
            endPoint = element;
          }
        }
      }

      var tempPositionList = [];
      let seriesDataList = op.series[0].data;
      for (let idx = 0; idx < seriesDataList.length; idx++) {
        let element = seriesDataList[idx];
        let y = element[1];
        if (y >= startPoint && y <= endPoint) {
          tempPositionList.push(element);
        }
      }

      if (tempPositionList.length == 0) {
        return;
      }

      let mergeDataList = mergeOp.series[0].data;

      if (mergeDataList == undefined) {
        return;
      }

      if (mergeDataList.length == 0) {
        mergeDataList = tempPositionList;
        mergeOp.series[0].data = mergeDataList;
      } else {
        let mergeStart = mergeDataList[0];
        let mergeLast = mergeDataList[mergeDataList.length - 1];
        let tempPositionStart = tempPositionList[0];
        let tempPositionLast = tempPositionList[tempPositionList.length - 1];

        let isMergeDown = mergeStart[1] > tempPositionLast[1];
        let isTmpDown = tempPositionStart[1] > mergeLast[1];

        if (isMergeDown || isTmpDown) {
          if (isMergeDown) {
            this.minPosition = tempPositionLast;
            this.maxPosition = mergeStart;
          } else {
            this.minPosition = mergeLast;
            this.maxPosition = tempPositionStart;
          }
          this.allowDomId = mergedomId;
          let opt = mergeChart.getOption();
          let mergSeries = opt.series[0];
          mergSeries.data = tempPositionList;
          mergeOp.series.push(mergSeries);

          let seriesJson = JSON.stringify(mergeOp.series[0]);
          let seriesData = JSON.parse(seriesJson);
          seriesData.symbol = "none";
          seriesData.data = [this.minPosition];
          mergeOp.series.push(seriesData);
          this.registerEvent();
        } else {
          mergeDataList = this.joinSort(mergeDataList, tempPositionList);
          mergeOp.series[0].data = mergeDataList;
        }
      }

      mergeData.option = mergeOp;
      mergeChart.setOption(mergeOp);
      this.chartList[mergeIndex] = mergeData;
    },
    clickPixel(pointInPixel) {
      let that = this;
      for (let index = 0; index < this.domList.length; index++) {
        const element = this.domList[index];
        let chart = that.chartObjs[element];
        if (chart == undefined) {
          continue;
        }
        let chartData = that.chartList[index];
        if (chartData == undefined) {
          continue;
        }
        let op = chartData.option;
        if (op == undefined) {
          continue;
        }
        let pointInGrid = chart.convertFromPixel(
          { seriesIndex: 0 },
          pointInPixel
        );
        let yIndex = pointInGrid[1];
        if (op.series.length == 0) {
          continue;
        }
        if (op.series[0] == undefined) {
          continue;
        }
        if (op.series[0].markLine != undefined) {
          op.series[0].markLine.data.push({
            yAxis: yIndex,
            symbol: "none",
          });
        }
        that.chartList[index].option = op;
        chart.setOption(op);
      }
    },
    addChart(dataList) {
      for (let index = 0; index < dataList.length; index++) {
        let element = dataList[index];
        const domIdx = this.domList.indexOf(element.chartId);
        if (domIdx == -1) {
          this.domList.push(element.chartId);
        }
        const chartIdx = this.chartList.findIndex((data) => {
          return data.chartId == element.chartId;
        });
        element.curveDesc = [
          {
            curveName: element.roadOption.curveName,
            rate: element.roadOption.rate,
          },
        ];
        if (chartIdx == -1) {
          this.chartList.push(element);
        } else {
          this.chartList[chartIdx] = element;
        }
      }
      this.$nextTick(() => {
        this.updateChartView();
      });
    },
    updataChart(dataList) {
      for (let index = 0; index < dataList.length; index++) {
        let element = dataList[index];
        const domIdx = this.domList.indexOf(element.chartId);
        if (domIdx == -1) {
          this.domList.push(element.chartId);
        }
        const chartIdx = this.chartList.findIndex((data) => {
          return data.chartId == element.chartId;
        });
        element.curveDesc = [
          {
            curveName: element.roadOption.curveName,
            rate: element.roadOption.rate,
          },
        ];
        if (chartIdx == -1) {
          this.chartList.push(element);
        } else {
          this.chartList[chartIdx] = element;
        }
      }
      let that = this;
      this.$nextTick(() => {
        this.updateChartView();
      });
    },
    deleteChart(dataList) {
      for (let index = 0; index < dataList.length; index++) {
        const element = dataList[index];
        const domIdx = this.domList.indexOf(element);
        if (domIdx != -1) {
          this.domList.splice(domIdx);
        }
        const chartIdx = this.chartList.find((data) => {
          return data.chartId == element;
        });
        if (chartIdx == -1) {
          this.chartList.splice(chartIdx);
        }

        if (this.chartObjs[element] != undefined) {
          this.chartObjs[element] = undefined;
        }
      }

      this.updateChartView();
    },
    updateChartView() {
      for (let index = 0; index < this.chartList.length; index++) {
        const element = this.chartList[index];
        const domId = element.chartId;
        const opt = element.option;
        if (domId == undefined || opt == undefined) {
          continue;
        }
        if (this.chartObjs[domId] == undefined) {
          let dom = document.getElementById(domId);
          if (dom == undefined) {
            continue;
          }
          this.chartObjs[domId] = markRaw(echarts.init(dom));
        }
        this.chartObjs[domId].clear();
        this.chartObjs[domId].setOption(opt);
      }
      this.addLister();
    },
    changeChart(oldIndex, newIndex, isChange) {
      let oldDomId = this.domList[oldIndex];
      if (oldDomId != undefined) {
        if (this.chartObjs[oldDomId] == undefined) {
          let dom = document.getElementById(oldDomId);
          if (dom == undefined) {
            return;
          }
          this.chartObjs[oldDomId] = markRaw(echarts.init(dom));
        }

        let oldData = this.chartList[oldIndex];
        if (oldData != undefined) {
          this.chartObjs[oldDomId].clear();
          this.chartObjs[oldDomId].setOption(oldData.option);
        }
      }

      let newDomId = this.domList[newIndex];
      if (newDomId != undefined) {
        if (this.chartObjs[newDomId] == undefined) {
          let dom = document.getElementById(newDomId);
          if (dom == undefined) {
            return;
          }
          this.chartObjs[newDomId] = markRaw(echarts.init(dom));
        }
        let newData = this.chartList[newIndex];
        if (newData != undefined) {
          // if (isChange) {
          this.chartObjs[newDomId].clear();
          // }
          this.chartObjs[newDomId].setOption(newData.option);
        }
      }
    },

    sendMessageToNative(event, data) {
      if (window.chrome == undefined) {
        console.log("获取不到window.chrome");
        return;
      }
      if (window.chrome.webview == undefined) {
        console.log("获取不到window.chrome.webview");
        return;
      }
      window.chrome.webview.postMessage({ event: event, data: data });
    },
  },
};
</script>

<style scoped>
::-webkit-scrollbar {
  width: 0 !important;
}

::-webkit-scrollbar {
  width: 0 !important;
  height: 0;
}

.container {
  width: 100%;
  margin: 0;
  background-color: #fff;
  display: flex;
  flex-direction: row;
  overflow: auto;
}

.mask_container {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

.chart_container {
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  /* width: 200px; */
  border-bottom: 1 solid #000;
  flex-wrap: nowrap;
}

.listChart {
  flex-grow: 1;
  /* width: 200px; */
  /* height: calc(100vh * 2); */
  /* border: 1px solid #000; */
  border-right: 1px solid #000;
  padding-top: 92px;
}

.empty_header {
  font-size: 12;
}

.list_header {
  /* width: 200px; */
  /* border: 0.5px solid #000; */
  position: fixed;
  top: 0;
  border-right: 1px solid #000;
  border-bottom: 1px solid #000;
  background-color: #fff;
  z-index: 999;
}

.list_header_title {
  font-size: 14px;
  color: #000;
  /* width: 100%; */
  text-align: center;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.list_header_empty {
  color: #fff;
}

.list_header_rate {
  /* margin-left: 10px;
  margin-right: 10px; */
  padding-left: 10px;
  padding-right: 10px;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
}

.list_header_line {
  height: 30px;
}

.list_header_bottom_line {
  border-bottom: 1px solid #000;
}

.list_header_empty_line {
  height: 60px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.chart_normal {
  border: 1px solid #000;
}

.chart_first {
  border-left: 1px solid #000;
  border-right: 1px solid #000;
}

:deep(.ant-table-tbody > tr > td) {
  padding: 0px;
  margin: 0;
}

:deep(.ant-table-thead > tr > th) {
  padding: 0px;
  margin: 0;
}

#wrapper {
  width: 100%;
  height: 100%;
}
</style>
