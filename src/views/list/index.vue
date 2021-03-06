<template>
  <div>
    <m-header :hasBack="hasBack" :titleTxt="titleTxt"></m-header>
    <m-content>
      <!--日历-->
      <div class="calendar">
        <div class="calendar__header">
          <div class="current__year">{{ selectedYear }}年</div>
          <div class="current__month swiper-container" id="topNav">
            <div class="swiper-wrapper scroll__month__body">
              <div
                class="swiper-slide scroll__item"
                :class="{ cur__month: i === selectedMonth }"
                v-for="(item, i) in calendarMonth"
                :data-month="item.mid"
                :style="{ width: navItemWidth }"
                :key="i"
              >{{ item.name }}</div>
            </div>
          </div>
          <div class="toogle__icon"></div>
        </div>
        <div class="calendar__main">
          <div class="main__head" v-for="(item, index) in calendarWeek" :key="index">{{ item }}</div>
          <div
            class="main__block"
            :class="{
              main__block__disabled: item.type !== 'nowMonth',
              main__block__today__selected:
                item.isToday && item.day === selectedDate,
              main__block__today: item.isToday,
              main__block__refunded: item.refund === 'REFUNDED',
              main__block__refunding: item.refund === 'REFUNDING',
              main__block__selected:
                item.day === selectedDate && item.type === 'nowMonth'
            }"
            @click.prevent="handleDayClick(item)"
            v-for="item in CALENDAR"
            :key="item.type + item.day"
          >{{ item.day }}</div>
        </div>
      </div>
    </m-content>
    <m-footer></m-footer>
  </div>
</template>

<script type="text/ecmascript-6">
import axios from "axios";
import Swiper from "swiper/dist/js/swiper.js"; //reason => https://github.com/nolimits4web/swiper/issues/2263
import timeUtil from "../../assets/js/utils/calendar";
export default {
  name: "listPage",
  data() {
    return {
      hasBack: false,
      titleTxt: "日历",
      calendarWeek: ["日", "一", "二", "三", "四", "五", "六"],
      calendarMonth: [
        { mid: "1", name: "1月" },
        { mid: "2", name: "2月" },
        { mid: "3", name: "3月" },
        { mid: "4", name: "4月" },
        { mid: "5", name: "5月" },
        { mid: "6", name: "6月" },
        { mid: "7", name: "7月" },
        { mid: "8", name: "8月" },
        { mid: "9", name: "9月" },
        { mid: "10", name: "10月" },
        { mid: "11", name: "11月" },
        { mid: "12", name: "12月" }
      ],
      CALENDAR: [],
      today: "", //今天
      currentDate: "", //服务器当前时间 2019-02-20
      selectedFullDate: "", //手动选择的日期
      selectedYear: "",
      selectedMonth: "",
      selectedDate: "",
      mySwiper: null,
      navItemWidth: "115px"
    };
  },
  mounted() {
    this.$nextTick(() => {
      this.initScroll();
      this.getMonthDays(this.selectedYear, this.selectedMonth);
    });
  },
  methods: {
    _map(arr1, arr2) {
      //arr1 ==> arr2
      arr1.map(k => {
        if (k.date) {
          k.date = k.date.replace(/-/g, "/");
        }
        return k;
      });
      const obj = {};
      arr1.map(e => {
        obj[e.date] = e.refund;
      });
      for (let i = 0; i < arr2.length; i++) {
        if (obj[arr2[i].date] !== undefined && arr2[i].type === "nowMonth") {
          arr2[i].refund = obj[arr2[i].date];
        }
      }
    },
    initScroll() {
      if (this.mySwiper) {
        this.mySwiper.destroy(true, false);
      }
      this.mySwiper = new Swiper("#topNav", {
        slidesPerView: "auto",
        freeMode: true,
        freeModeMomentumRatio: 0.5,
        observer: true,
        observeParents: false,
        on: {
          init: () => {
            //默认选中
            this.initData();
          },
          tap: () => {
            //滑动时间
            this.mySwiper.setTransition(300);
            //滑动
            this._slide();
            //更改class
            this.selectedMonth = this.mySwiper.clickedIndex;

            this.getMonthDays(this.selectedYear, this.selectedMonth);
          },
          touchmove: e => {
            //解决拖动警告
            e.stopPropagation();
            e.preventDefault();
          }
        }
      });
    },
    _slide() {
      //点击的nav
      const slide = this.mySwiper.slides[this.mySwiper.clickedIndex];
      //点击的nav offsetLeft距离浏览器左边距离
      const slideLeft = slide.offsetLeft;
      //点击的nav的可视宽度
      const slideWidth = slide.clientWidth;
      //导航可视宽度
      const clientWidth = this.mySwiper.width;
      //导航总宽度
      const swiperWidth = slideWidth * this.mySwiper.slides.length;
      if (slideLeft < (clientWidth - parseInt(slideWidth)) / 2) {
        this.mySwiper.setTranslate(0);
      } else if (
        slideLeft >
        swiperWidth - parseInt(slideWidth + clientWidth) / 2
      ) {
        this.mySwiper.setTranslate(clientWidth - swiperWidth);
      } else {
        this.mySwiper.setTranslate(
          (clientWidth - parseInt(slideWidth)) / 2 - slideLeft
        );
      }
    },
    initData() {
      this.today = this.currentDate || timeUtil.getDateStr(0); // 如果没有服务器时间，拿本地时间
      // 是否有手动选中的日期
      this.selectedFullDate = this.currentDate || timeUtil.getDateStr(0);
      this.selectedYear = Number(this.selectedFullDate.split("-")[0]);
      this.selectedMonth = Number(this.selectedFullDate.split("-")[1]) - 1;
      this.selectedDate = Number(this.selectedFullDate.split("-")[2]);
      this.selectedFullDate = `${this.selectedYear}-${timeUtil._toDou(
        this.selectedMonth + 1
      )}-${timeUtil._toDou(this.selectedDate)}`;
    },
    _filter(year, month) {
      return new Promise(resolve => {
        window.CALENDAR = timeUtil.getMonthList(new Date(year, month));
        axios
          .get(" https://easy-mock.com/mock/5c4137f5cee47f2c67974e24/api/dates")
          .then(res => {
            const data = res.data;
            if (data.code === 0) {
              this._map(data.data, window.CALENDAR);
              resolve(window.CALENDAR);
            }
          })
          .catch(() => {
            resolve(window.CALENDAR);
          });
      });
    },
    //渲染日期
    async getMonthDays(year, month) {
      this.CALENDAR = await this._filter(year, month);
    },
    handleDayClick(item) {
      if (item.type === "nowMonth") {
        // do anything...
        this.selectedDate = Number(item.day);
      }
    },
    handlePreMonth() {
      if (this.prevYearMonthBoolean) return;
      if (this.selectedMonth === 0) {
        this.selectedYear = this.selectedYear - 1;
        this.selectedMonth = 11;
        this.selectedDate = 1;
      } else {
        this.selectedMonth = this.selectedMonth - 1;
        this.selectedDate = 1;
      }
    },
    handleNextMonth() {
      if (this.nextYearMonthBoolean) return;
      if (this.selectedMonth === 11) {
        this.selectedYear = this.selectedYear + 1;
        this.selectedMonth = 0;
        this.selectedDate = 1;
      } else {
        this.selectedMonth = this.selectedMonth + 1;
        this.selectedDate = 1;
      }
    }
  }
};
</script>
<style lang="scss" scoped>
.calendar {
  flex-shrink: 0;
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 15px 0 20px 0;
  border-radius: 4px;
  background-color: white;
  box-shadow: 0 0 10px rgba(208, 208, 208, 0.5);
}
.calendar__header {
  color: #2c3135;
  font-size: 16px;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 90px;
  overflow: hidden;
  box-shadow: 0px 8px 24px 0px rgba(225, 227, 237, 0.5);
  .current__year {
    padding-left: 30px;
    flex: 0 0 150px;
    font-size: 28px;
  }
  .current__month {
    flex: 0 0 520px;
    position: relative;
    overflow-x: hidden;
    .scroll__month__body {
      width: 100%;
      height: 100%;
      display: flex;
      white-space: nowrap;
      -webkit-user-drag: none;
      touch-action: pan-x;
      touch-action: pan-y;
    }
    .scroll__item {
      flex: 1;
      flex-grow: 1;
      flex-shrink: 0;
      flex-basis: auto;
      box-sizing: border-box;
      font-size: 28px;
      text-align: center;
      color: #000;
      display: block;
      &.cur__month {
        color: #f90;
        font-weight: 550;
      }
    }
  }
  .toogle__icon {
    flex: 0 0 60px;
    height: 30px;
    background: url(./images/list.png) no-repeat center 0;
    background-size: 30px auto;
    font-size: 30px;
  }
}
.calendar__main {
  width: 700px;
  display: flex;
  justify-content: space-around;
  flex-wrap: wrap;
  margin: 10px 0 15px 0;
  .main__head {
    width: 80px;
    height: 80px;
    margin: 10px;
    border-radius: 2px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 26px;
    color: #7f8fa4;
    background-color: #fff;
    flex-shrink: 0;
  }
  .main__block {
    width: 80px;
    height: 80px;
    margin: 10px;
    border-radius: 50%;
    font-size: 25px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #2e2e39;
    background-color: #fff;
    flex-shrink: 0;
    box-shadow: 0;
    position: relative;
    transition: 0.3s all;
    &::after {
      display: none;
      content: "";
      width: 10px;
      height: 10px;
      border-radius: 50%;
      position: absolute;
      bottom: 15px;
      left: 50%;
      transform: translateX(-50%);
    }
    &.main__block__disabled {
      color: #cbc9d5;
    }
    &.main__block__refunded {
      &::after {
        display: block;
        background: red;
      }
    }
    &.main__block__refunding {
      &::after {
        display: block;
        background: #ccc;
      }
    }
    &.main__block__selected {
      background-color: #333;
      color: #fff;
    }
    &.main__block__today__selected {
      color: #fff !important;
      background: #ff8200 !important;
    }
    &.main__block__today {
      color: #ff8200;
      background: #fff;
    }
  }
}
</style>
