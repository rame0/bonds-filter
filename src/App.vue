<template>
  <div class="container-fluid">
    <div class="row">
      <div class="col-9">
        <div class="alert alert-info" v-if="isLoading">
          Идет обработка запроса.
          Обработано {{ updateProgress }} строк из {{ total }}.
        </div>
        <BondsTable :rows="rows" :columns="columns" v-model:sort="sort"></BondsTable>
      </div>
      <div class="col-3">
        <h6>Доход (от - до)</h6>
        <div class="row">
          <div class="col">
            <input type="number" min="0" :max="form.YieldLess-1" step="0.01" id="YieldMore" v-model="form.YieldMore"
                   class="form-control" :disabled="isLoading">
          </div>
          <div class="col">
            <input type="number" :min="form.YieldMore+1" max="300" step="0.01" id="YieldLess"
                   v-model="form.YieldLess"
                   class="form-control" :disabled="isLoading">
          </div>
        </div>

        <h6>Цена (от - до)</h6>
        <div class="row">
          <div class="col">
            <input type="number" min="0" :max="form.PriceLess-1" id="PriceMore" v-model="form.PriceMore"
                   class="form-control" :disabled="isLoading">
          </div>
          <div class="col">
            <input type="number" :min="form.PriceMore+1" max="300" id="PriceLess" v-model="form.PriceLess"
                   class="form-control" :disabled="isLoading">
          </div>
        </div>
        <h6>Дюрация (от - до)</h6>
        <div class="row">
          <div class="col">
            <input type="number" min="0" :max="form.DurationLess-1" id="DurationMore" v-model="form.DurationMore"
                   class="form-control" :disabled="isLoading">
          </div>
          <div class="col">
            <input type="number" :min="form.DurationMore+1" max="300" id="DurationLess" v-model="form.DurationLess"
                   class="form-control" :disabled="isLoading">
          </div>
        </div>
<!--        <h6>Объем сделок</h6>-->
<!--        <input type="number" min="0" id="VolumeMore" v-model="form.VolumeMore" class="form-control"-->
<!--               :disabled="isLoading">-->
<!---->
<!--        <h6>Совокупный объем сделок</h6>-->
<!--        <input type="number" min="0" id="BondVolumeMore" v-model="form.BondVolumeMore" class="form-control"-->
<!--               :disabled="isLoading">-->

        <h6>Учитывать, чтобы денежные выплаты были известны до самого погашения?</h6>
        <input type="checkbox" name="OfferYesNo" :checked="form.OfferYesNo" :disabled="isLoading">
        <div class="row">
          <div class="col">
            Показано <b>{{ filtered }}</b> из <u>{{ total }}</u>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import {computed, ref} from "vue";
import moment from "moment"
import BondsTable from "@/components/Table/BondsTable";

/**
 *
 * @type {*[{BondName, SECID, BondPrice, BondVolume, BondYield, BondDuration, MonthsOfPaymentsDates}]}
 */
let marketData = []

export default {
  name: 'App',
  components: {BondsTable},
  data () {
    return {
      form: {
        YieldMore: 20,
        YieldLess: 40,
        PriceMore: 60,
        PriceLess: 110,
        DurationMore: 6,
        DurationLess: 13,
        VolumeMore: 400,
        BondVolumeMore: 10000,
        OfferYesNo: true,
      },
      rows: [],
      total: 0,
      filtered: 0,
      isLoading: false,
      updateProgress: 0,
    }
  },
  watch: {
    form: {
      deep: true,
      handler (newValue) {
        this.isLoading = true;
        this.filterAndShow(newValue)
      }
    }
  },
  setup () {
    const convertUnit = (unit) => {
      switch (unit) {
        case 'SUR':
          return '₽'
        case 'USD':
          return '$'
        case 'EUR':
          return '$'
        case 'CNY':
          return '¥'
        default:
          return '??'
      }
    }

    const columns = computed(() =>
        [
          {name: "BondName", title: "Полное наименование", field: "BondName", sortable: true},
          {name: "SECID", title: "Код ценной бумаги", field: "SECID", sortable: true},
          {name: "BondPrice", title: "Цена, %", field: "BondPrice", sortable: true},
          {
            name: "BondVolume", title: "Объем сделок", field: "BondVolume", sortable: true,
            compute: (row) => `${row.BondVolume} шт за ${row.tradeDays} дней`
          },
          {name: "BondYield", title: "Доходность", field: "BondYield", sortable: true,},
          {name: "BondDuration", title: "Дюрация, месяцев", field: "BondDuration", sortable: true,},
          {name: "CouponPeriod", title: "Пер. купона", field: "CouponPeriod", sortable: true,},
          {name: "MonthsOfPaymentsDates", title: "Месяцы выплат", field: "MonthsOfPaymentsDates", sortable: true,},
          {
            name: "lowLiquidity", title: "Ликвидность", field: "lowLiquidity", sortable: true,
            compute: (row) => row.lowLiquidity ? "Низкая" : "Высокая"
          },
          {
            name: "FaceValue", title: "Номинал", field: "FaceValue", sortable: true,
            compute: (row) => `${row.FaceValue} ${convertUnit(row.FaceUnit)}`
          },

        ],
    )

    const sort = ref({column: "BondName", descending: true})


    return {columns, sort, convertUnit}
  },
  mounted () {
    this.isLoading = true;
    const sec_request_columns = "SECID,SECNAME,PREVLEGALCLOSEPRICE,BOARDID,COUPONPERIOD,FACEVALUE,FACEUNIT";
    let url1 = `https://iss.moex.com/iss/engines/stock/markets/bonds/boardgroups/7/securities.json?iss.dp=comma&iss.meta=off&iss.only=securities,marketdata&securities.columns=${sec_request_columns}&marketdata.columns=SECID,YIELD,DURATION`
    let url2 = `https://iss.moex.com/iss/engines/stock/markets/bonds/boardgroups/58/securities.json?iss.dp=comma&iss.meta=off&iss.only=securities,marketdata&securities.columns=${sec_request_columns}&marketdata.columns=SECID,YIELD,DURATION`
    let url3 = `https://iss.moex.com/iss/engines/stock/markets/bonds/boardgroups/193/securities.json?iss.dp=comma&iss.meta=off&iss.only=securities,marketdata&securities.columns=${sec_request_columns}&marketdata.columns=SECID,YIELD,DURATION`

    axios.all([
      axios.get(url1),
      axios.get(url2),
      axios.get(url3)
    ])
        .then(axios.spread((firstResponse, secondResponse, thirdResponse) => {
          // console.log(firstResponse.data)
          // throw new Error()
          marketData = marketData.concat(this.buildDataFromResponse(firstResponse.data));
          marketData = marketData.concat(this.buildDataFromResponse(secondResponse.data));
          marketData = marketData.concat(this.buildDataFromResponse(thirdResponse.data));

          this.total = marketData.length;
          this.filterAndShow(this.form)
        }))
        .catch(error => console.log(error));
  },
  methods: {
    buildDataFromResponse (responce) {
      let result = [];
      for (let i = 0; i < responce.securities.data.length; i++) {
        result.push({
          BondName: responce.securities.data[i][1].replace(/"/g, '').replace(/'/g, ''),
          SECID: responce.securities.data[i][0],
          BOARDID: responce.securities.data[i][3],
          FaceValue: responce.securities.data[i][5],
          FaceUnit: responce.securities.data[i][6],
          BondPrice: responce.securities.data[i][2],
          CouponPeriod: responce.securities.data[i][4],
          BondVolume: -1,
          BondYield: responce.marketdata.data[i][1],
          BondDuration: Math.floor((responce.marketdata.data[i][2] / 30) * 100) / 100,
          MonthsOfPaymentsDates: null,
          lowLiquidity: false,
        });
      }
      return result;
    },
    async filterAndShow (filters, thresholdValue = 1) {
      const DateRequestPrevious = moment().subtract(15, 'days').format('YYYY-MM-DD') // `${now.getFullYear()}-${now.getMonth() + 1}-${now.getDate() - 15}`; //этот день n дней назад

      let filteredData = [];

      this.updateProgress = 0;

      this.rows = [];
      for (let i = 0; i < marketData.length; i++) {
        this.updateProgress++;
        if (marketData[i].BondYield > filters.YieldMore && marketData[i].BondYield < filters.YieldLess
            && marketData[i].BondPrice > filters.PriceMore && marketData[i].BondPrice < filters.PriceLess
            && marketData[i].BondDuration > filters.DurationMore && marketData[i].BondDuration < filters.DurationLess) {
          if (!marketData[i].MonthsOfPaymentsDates) {
            const coupons_data = await axios.get(`https://iss.moex.com/iss/statistics/engines/stock/markets/bonds/bondization/${marketData[i].SECID}.json?iss.meta=off&iss.only=coupons`);
            let couponDates = []
            let value_rubNull = 0
            for (let j = 0; j <= coupons_data.data.coupons.data.length - 1; j++) {
              let coupondate = coupons_data.data.coupons.data[j][3] // даты купона
              let value_rub = coupons_data.data.coupons.data[j][9] // сумма выплаты купона
              let inFuture = new Date(coupondate) > new Date()
              if (inFuture === true) {
                couponDates.push(+coupondate.split("-")[1])
                if (value_rub == null) {
                  value_rubNull += 1
                }
              }

              if(value_rubNull > 0 && filters.OfferYesNo){
                continue
              }

              let uniqueDates = [...new Set(couponDates)] // уникальные значения месяцев
              uniqueDates = uniqueDates.sort(function (a, b) {
                return a - b;
              })

              let formattedDates = ''
              for (let y = 1; y < 13; y++) {
                formattedDates += uniqueDates.includes(y) ? `${y} ` : ` `
              }
              marketData[i].MonthsOfPaymentsDates = formattedDates
                  .replace(/1\s/g, 'янв ')
                  .replace(/2\s/g, 'фев ')
                  .replace(/3\s/g, 'мар ')
                  .replace(/4\s/g, 'апр ')
                  .replace(/5\s/g, 'май ')
                  .replace(/6\s/g, 'июн ')
                  .replace(/7\s/g, 'июл ')
                  .replace(/8\s/g, 'авг ')
                  .replace(/9\s/g, 'сен ')
                  .replace(/10\s/g, 'окт ')
                  .replace(/11\s/g, 'ноя ')
                  .replace(/12\s/g, 'дек ')
            }
          }
          if (marketData[i].BondVolume < 0) {
            const trade_data = await axios.get(`https://iss.moex.com/iss/history/engines/stock/markets/bonds/boards/${marketData[i].BOARDID}/securities/${marketData[i].SECID}.json?iss.meta=off&iss.only=history&history.columns=SECID,TRADEDATE,VOLUME,NUMTRADES&limit=20&from=${DateRequestPrevious}`)
            if (!trade_data.data.history) {
              marketData[i].BondVolume = 0
              marketData[i].lowLiquidity = true
              continue
            }
            let volume_sum = 0
            let lowLiquidity = false

            for (let hist_item of trade_data.data.history.data) {
              const volume = hist_item[2];
              volume_sum += volume;
              if (thresholdValue > volume) { // если оборот в конкретный день меньше
                lowLiquidity = true
              }
            }
            marketData[i].tradeDays = trade_data.data.history.data.length;
            if (marketData[i].tradeDays < 6) { // если всего дней в апи на этом периоде очень мало
              lowLiquidity = true
            }

            marketData[i].lowLiquidity = lowLiquidity
            marketData[i].BondVolume = volume_sum

          }


          filteredData.push(marketData[i]);
        }
      }
      this.rows = filteredData;
      this.filtered = filteredData.length;
      this.isLoading = false;
    },
  }
}
</script>
