<template>
  <div class="dashboard-wrapper">
    <div class="dashboard-header mb-4">
      <h1 class="h1 font-weight-bold mb-0">PD PARAWOOD Dashboard</h1>
      <p class="timestamp text-muted small">อัปเดตล่าสุด: {{ currentTime }}</p>
    </div>

    <div class="stats-grid mb-5">
      <div class="stat-card">
        <div class="card-indicator purple"></div>
        <div class="stat-body">
          <div class="stat-header">
            <h3>เป้าหมายรายวัน</h3>
          </div>
          <p class="stat-value">{{ dailyTarget.toLocaleString() }}</p>
          <p class="stat-label text-muted">เป้าหมายที่ตั้งไว้ (บาท)</p>
        </div>
      </div>

      <div class="stat-card">
        <div class="card-indicator"></div>
        <div class="stat-body">
          <div class="stat-header">
            <h3>ผลิตได้</h3>
          </div>
          <p class="stat-value">{{ alreadyProduced.toLocaleString() }}</p>
          <div class="status-badge" :class="alreadyProduced >= dailyTarget ? 'bg-light-success' : 'bg-light-danger'">
            {{ alreadyProduced >= dailyTarget ? '✓ ได้ตามเป้า' : '✗ ยังไม่ถึงเป้า' }}
          </div>
        </div>
      </div>

      <div class="stat-card">
        <div class="card-indicator" :class="(alreadyProduced - dailyTarget) >= 0 ? 'green' : 'red'"></div>
        <div class="stat-body">
          <div class="stat-header">
            <h3>ส่วนต่างจากเป้าหมาย</h3>
          </div>
          <p class="stat-value" :class="(alreadyProduced - dailyTarget) >= 0 ? 'text-success' : 'text-danger'">
            {{ (alreadyProduced - dailyTarget).toLocaleString() }}
          </p>
          <p class="stat-label text-muted">เปรียบเทียบจากเป้าหมายรายวัน</p>
        </div>
      </div>
    </div>

    <div class="section-title-wrapper mb-3">
      <h2 class="section-title">สรุปสต็อกจำนวนสินค้า (Stock Summary)</h2>
      <div class="section-underline"></div>
    </div>

    <div class="search-card shadow-sm mb-4 bg-white p-4" style="border-radius: 12px;">
      <div class="d-flex flex-wrap align-items-end">
        <div class="position-relative mr-3" style="flex: 1; max-width: 400px;">
          <label class="small text-muted font-weight-bold mb-2">ระบุรหัส SAP/FG code:</label>
          <input 
            type="text" 
            v-model="sapInput" 
            class="form-control" 
            placeholder="กรอกรหัส SAP/FG code..." 
            @focus="showSuggestions = true"
            @input="showSuggestions = true" 
            @blur="handleBlur"
            @keyup.enter="processDisplay"
          >
          <ul v-if="showSuggestions && filteredSuggestions.length > 0" class="custom-autocomplete shadow">
            <li v-for="(item, index) in filteredSuggestions" :key="index" @mousedown.prevent="selectSuggestion(item)" class="autocomplete-item">
              <span class="font-weight-bold" style="color: #7367f0;">{{ item.sap }}</span>
              <span v-if="item.prod" class="text-muted small ml-2">| {{ item.prod }}</span>
            </li>
          </ul>
        </div>
        <div class="d-flex">
          <button @click="processDisplay" class="btn btn-primary px-4 mr-3 custom-btn-search">search</button>
          <button @click="resetSearch" class="btn btn-outline-secondary px-4 custom-btn-seeall">see all</button>
        </div>
      </div>
    </div>

    <div class="position-relative mb-5" style="min-height: 400px;">
      <div v-if="isLoading" class="text-center py-5">
        <div class="spinner-border text-primary" role="status"></div>
        <p class="mt-3 font-weight-bold text-primary">Loading Charts...</p>
      </div>
      <div v-else>
        <div v-show="displayList.length > 0" class="chart-card shadow-sm p-4 bg-white border" style="border-radius: 12px; border-top: 5px solid #28c76f !important;">
          <h5 class="font-weight-bold text-success mb-4 text-center">
            {{ isSearching ? 'สรุปยอดสต็อกตามการค้นหา' : 'สรุปยอดสต็อกชิ้นส่วนสินค้าทั้งหมด (Total Summary)' }}
          </h5>
          <div class="row align-items-start">
            <div class="col-lg-6 col-md-12 border-right">
              <div ref="totalChartRef" style="width: 100%; height: 400px;"></div>
            </div>
            <div class="col-lg-6 col-md-12">
              <div class="table-responsive" style="max-height: 400px; overflow-y: auto;">
                <table class="table table-sm table-hover border-bottom">
                  <thead class="bg-light sticky-top" style="z-index: 1;">
                    <tr>
                      <th class="small font-weight-bold py-2">รายละเอียดสินค้า</th>
                      <th class="small font-weight-bold text-right py-2" style="width: 120px;">ชิ้น</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="(item, idx) in stockTableData" :key="idx">
                      <td class="small py-2">
                        <div class="font-weight-bold text-dark">{{ item.name }}</div>
                        <code class="text-muted" style="font-size: 0.75rem;">{{ item.code }}</code>
                      </td>
                      <td class="small py-2 text-right font-weight-bold text-dark">
                        {{ item.value.toLocaleString() }}
                      </td>
                    </tr>
                  </tbody>
                  <tfoot class="bg-light font-weight-bold">
                    <tr>
                      <td class="small py-2 text-center">ยอดสต็อกรวม</td>
                      <td class="small py-2 text-right text-success" style="font-size: 1rem;">
                        {{ stockChartData.reduce((sum, i) => sum + i.value, 0).toLocaleString() }}
                      </td>
                    </tr>
                  </tfoot>
                </table>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="section-title-wrapper mt-5 mb-4">
      <div class="d-flex justify-content-between align-items-end">
        <div>
          <h2 class="section-title mb-1">Production Yield (รายงานยอดผลิต)</h2>
          <p v-if="lastUpdate" class="text-muted small mb-0">อัปเดตล่าสุด: {{ lastUpdate }}</p>
        </div>
        <div class="text-right">
          <div class="mb-1">
            <span class="text-muted small font-weight-bold">รวมมูลค่าตามการกรอง</span>
            <h3 class="text-danger font-weight-bolder mb-0">{{ totalPriceFormatted }} บาท</h3>
          </div>
          <b-button variant="primary" @click="fetchAndSortProductionData" :disabled="yieldLoading" class="btn-retrieve shadow-sm">
            {{ yieldLoading ? 'Retrieving...' : 'Retrieve data' }}
          </b-button>
        </div>
      </div>
    </div>

    <div class="production-report-wrapper mb-5">
      <div class="filter-container p-3 mb-4 shadow-sm bg-white">
        <b-row>
          <b-col md="2">
            <label class="filter-label">ปี</label>
            <b-form-select v-model="filters.year" :options="yearOptions" @change="handleFilterChange" size="sm" class="custom-select-ui" />
          </b-col>
          <b-col md="2">
            <label class="filter-label">เดือน</label>
            <b-form-select v-model="filters.month" @change="handleFilterChange" size="sm" class="custom-select-ui">
              <option v-for="(m, index) in months" :key="index" :value="index + 1">{{ m }}</option>
            </b-form-select>
          </b-col>
          <b-col md="5">
            <label class="filter-label">เลือกช่วงสัปดาห์</label>
            <b-form-select v-model="filters.week" @change="handleFilterChange" size="sm" class="custom-select-ui">
              <option value="all">ดูข้อมูลทั้งเดือน</option>
              <option v-for="w in weekOptions" :key="w.value" :value="w.value">{{ w.text }}</option>
            </b-form-select>
          </b-col>
          <b-col md="3">
            <label class="filter-label">ระบุวันที่</label>
            <b-form-select v-model="filters.day" @change="handleFilterChange" size="sm" class="custom-select-ui">
              <option value="all">ทั้งหมด </option>
              <option v-for="d in dayOptions" :key="d" :value="d">{{ d }}</option>
            </b-form-select>
          </b-col>
        </b-row>
      </div>

      <div class="table-container shadow-sm bg-white">
        <div class="table-responsive">
          <table class="table table-custom mb-0">
            <thead>
              <tr class="header-row">
                <th style="width: 40px;">#</th>
                <th style="width: 70px;">เวลา</th>
                <th style="width: 90px;">วันที่</th>
                <th style="width: 120px;">ใบผลิต (MO)</th>
                <th style="width: 170px;">รหัสสินค้า</th>
                <th style="width: 150px;">รายละเอียดสินค้า</th>
                <th style="width: 110px;">สี</th>
                <th style="width: 70px;" class="text-right">จำนวน</th>
                <th style="width: 90px;" class="text-right">มูลค่า (บาท)</th>
              </tr>
              <tr class="filter-row">
                <th></th><th></th><th></th>
                <th><b-form-input v-model="colFilters.moNo" @input="currentPage = 1" size="sm" placeholder="กรอง MO..." class="table-filter-input" /></th>
                <th><b-form-input v-model="colFilters.pCode" @input="currentPage = 1" size="sm" placeholder="กรองรหัส..." class="table-filter-input" /></th>
                <th><b-form-input v-model="colFilters.pDesc" @input="currentPage = 1" size="sm" placeholder="ชื่อรายการ..." class="table-filter-input" /></th>
                <th>
                  <b-form-select v-model="colFilters.color" @change="currentPage = 1" size="sm" class="table-filter-input">
                    <option value="">ทั้งหมด</option>
                    <option v-for="c in colorOptions" :key="c" :value="c">{{ c }}</option>
                  </b-form-select>
                </th>
                <th></th><th></th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="yieldLoading" class="text-center"><td colspan="9" class="py-5">กำลังโหลดข้อมูล...</td></tr>
              <tr v-for="(item, index) in paginatedItems" :key="index" class="data-row">
                <td class="text-center text-muted border-right">{{ (filters.day === 'all' ? (currentPage - 1) * perPage : 0) + index + 1 }}</td>
                <td class="border-right">{{ item.displayTime }}</td>
                <td class="border-right text-dark">{{ item.displayDate }}</td>
                <td class="border-right">{{ item.moNo }}</td>
                <td class="border-right font-weight-bold text-dark">{{ item.pCode }}</td>
                <td class="border-right text-truncate desc-column" :title="item.pDesc">{{ item.pDesc }}</td>
                <td class="border-right">{{ item.color }}</td>
                <td class="border-right text-right font-weight-bold text-dark">{{ item.qty.toLocaleString() }}</td>
                <td class="text-right font-weight-bold text-dark">{{ item.priceStr }}</td>
              </tr>
            </tbody>
          </table>
        </div>
        <div v-if="filters.day === 'all'" class="d-flex justify-content-between align-items-center p-3 border-top bg-light">
          <div class="text-muted small">
            แสดง {{ Math.min((currentPage - 1) * perPage + 1, filteredYieldItems.length) }} - {{ Math.min(currentPage * perPage, filteredYieldItems.length) }} จาก {{ filteredYieldItems.length }} รายการ
          </div>
          <b-pagination v-model="currentPage" :total-rows="filteredYieldItems.length" :per-page="perPage" size="sm" class="mb-0" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as echarts from 'echarts';
import Papa from 'papaparse';
import axios from 'axios';
import { BRow, BCol, BButton, BFormSelect, BFormInput, BPagination } from 'bootstrap-vue';
import Ripple from 'vue-ripple-directive';

export default {
  name: 'FullProductionDashboard',
  components: { BRow, BCol, BButton, BFormSelect, BFormInput, BPagination },
  directives: { Ripple },
  data() {
    const now = new Date();
    return {
      dailyTarget: 20000, 
      alreadyProduced: 3120, 
      currentTime: '',
      URL_1: "https://docs.google.com/spreadsheets/d/e/2PACX-1vT-sDov5Hl1nWrRBz9jCiiBeatitpYjZp0fqPYngNmvdY1teoDLQrhPBNkMZATc-rktKV37X2Q1qzkm/pub?output=csv",
      URL_2: "https://docs.google.com/spreadsheets/d/e/2PACX-1vShU_0KepJ0kBsIYP6zPhKVuZEE4TbCDyvx81yzPMBbdtO3SJp1kf1bRc4hbFfs4ErULD0oDyK39CkE/pub?output=csv",
      YIELD_CSV_URL: 'https://docs.google.com/spreadsheets/d/e/2PACX-1vSpJBH3HzXDIWC0B_WuHdAfSJiyZPfMTVldJVcH9b4LNStGfZpUewPPEjfHIvIowGua2fhdOApJT1E4/pub?output=csv',
      sapInput: "", showSuggestions: false, isSearching: false, masterList: [], displayList: [], stockDictionary: {},
      totalChartInstance: null, isLoading: true, yieldLoading: false, lastUpdate: '', yieldItems: [],
      currentPage: 1, perPage: 15,
      filters: { 
        year: now.getFullYear(), 
        month: now.getMonth() + 1, 
        week: 'all', 
        day: 'all' 
      },
      colFilters: { moNo: '', pCode: '', pDesc: '', color: '' },
      months: ['มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน', 'พฤษภาคม', 'มิถุนายน', 'กรกฎาคม', 'สิงหาคม', 'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม'],
      yearOptions: [], dayOptions: [], weekOptions: []
    };
  },
  computed: {
    filteredSuggestions() {
      const searchTxt = this.sapInput.trim().toLowerCase();
      if (!searchTxt) return [];
      const uniqueMap = new Map();
      this.masterList.forEach(item => {
        if (item.sap && item.sap.toLowerCase().includes(searchTxt)) {
          if (!uniqueMap.has(item.sap)) uniqueMap.set(item.sap, { sap: item.sap, prod: item.prod });
        }
      });
      return Array.from(uniqueMap.values());
    },
    stockChartData() {
      const unique = []; const used = new Set();
      this.displayList.forEach(item => {
        if (!used.has(item.code)) { unique.push(item); used.add(item.code); }
      });
      return unique.sort((a, b) => b.value - a.value);
    },
    stockTableData() { return this.stockChartData.slice(0, 20); },
    filteredYieldItems() {
      return this.yieldItems.filter(item => (
        item.moNo.toLowerCase().includes(this.colFilters.moNo.toLowerCase()) &&
        item.pCode.toLowerCase().includes(this.colFilters.pCode.toLowerCase()) &&
        item.pDesc.toLowerCase().includes(this.colFilters.pDesc.toLowerCase()) &&
        (this.colFilters.color === '' || item.color === this.colFilters.color)
      ));
    },
    paginatedItems() {
      if (this.filters.day !== 'all') return this.filteredYieldItems;
      const start = (this.currentPage - 1) * this.perPage;
      return this.filteredYieldItems.slice(start, start + this.perPage);
    },
    totalPriceFormatted() {
      const total = this.filteredYieldItems.reduce((sum, item) => sum + (parseFloat(item.priceStr.replace(/,/g, '')) || 0), 0);
      return total.toLocaleString('th-TH', { minimumFractionDigits: 2 });
    },
    colorOptions() { return [...new Set(this.yieldItems.map(i => i.color).filter(v => v && v !== '-'))].sort(); }
  },
  async mounted() {
    this.updateTime(); setInterval(this.updateTime, 1000);
    await this.fetchData(); this.setupYearSelect(); this.updateYieldOptions(); this.fetchAndSortProductionData();
    window.addEventListener('resize', this.handleResize);
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.handleResize);
    if (this.totalChartInstance) {
      this.totalChartInstance.dispose();
    }
  },
  methods: {
    updateTime() { this.currentTime = new Date().toLocaleString('th-TH'); },
    handleFilterChange() { this.currentPage = 1; this.updateYieldOptions(); this.fetchAndSortProductionData(); },
    async fetchData() {
      this.isLoading = true;
      try {
        const [csv1, csv2] = await Promise.all([fetch(this.URL_1).then(r => r.text()), fetch(this.URL_2).then(r => r.text())]);
        const stockRows = Papa.parse(csv2).data;
        stockRows.slice(1).forEach(row => { if (row[1]) this.stockDictionary[row[1].trim()] = parseFloat(String(row[7]).replace(/,/g, '')) || 0; });
        let lSap = "", lProd = "";
        this.masterList = Papa.parse(csv1).data.slice(1).map(row => {
          if (row[0]?.trim()) lSap = row[0].trim(); if (row[1]?.trim()) lProd = row[1].trim();
          return { sap: lSap, prod: lProd, code: row[2]?.trim(), name: row[3]?.trim(), value: this.stockDictionary[row[2]?.trim()] || 0 };
        }).filter(i => i.code);
        this.displayList = [...this.masterList]; this.renderAllCharts();
      } catch (e) { console.error(e); } finally { this.isLoading = false; }
    },
    processDisplay() {
      const s = this.sapInput.trim().toLowerCase();
      this.displayList = s ? this.masterList.filter(i => i.sap.toLowerCase() === s) : this.masterList;
      this.isSearching = !!s; this.renderAllCharts();
    },
    renderAllCharts() { 
      if (this.totalChartInstance) this.totalChartInstance.dispose(); 
      setTimeout(() => this.initTotalChart(), 200); 
    },
    initTotalChart() {
      if (!this.$refs.totalChartRef) return;
      this.totalChartInstance = echarts.init(this.$refs.totalChartRef);
      this.totalChartInstance.setOption({
        animationDuration: 1000,
        tooltip: { trigger: 'item' }, 
        legend: { bottom: 0, type: 'scroll' },
        series: [{ 
          type: 'pie', 
          radius: ['40%', '70%'], 
          data: this.stockChartData, 
          label: { show: false },
          itemStyle: { borderRadius: 8, borderColor: '#fff', borderWidth: 2 }
        }]
      });
    },
    setupYearSelect() { const c = new Date().getFullYear(); this.yearOptions = [c - 1, c, c + 1]; },
    updateYieldOptions() {
      const y = this.filters.year, m = this.filters.month - 1, last = new Date(y, m + 1, 0).getDate();
      this.dayOptions = Array.from({ length: last }, (_, i) => i + 1);
      this.weekOptions = []; let curr = new Date(y, m, 1), w = 1;
      while (curr.getMonth() === m) {
        let start = curr.getDate(), end = Math.min(start + (6 - curr.getDay()), last);
        this.weekOptions.push({ value: `${start}|${end}`, text: `สัปดาห์ที่ ${w} (${start}/${m+1})` });
        curr.setDate(end + 1); w++;
      }
    },
    async fetchAndSortProductionData() {
      this.yieldLoading = true;
      try {
        const res = await axios.get(`${this.YIELD_CSV_URL}&t=${Date.now()}`);
        let ds = 1, de = 31;
        if (this.filters.day !== 'all') ds = de = parseInt(this.filters.day);
        else if (this.filters.week !== 'all') [ds, de] = this.filters.week.split('|').map(Number);
        this.yieldItems = Papa.parse(res.data).data.slice(1).map(cols => {
          const dt = cols[0]?.split(' ') || []; const dp = dt[0]?.split('/') || [];
          const yRaw = parseInt(dp[2]); const y = yRaw < 2000 ? yRaw + 2000 : (yRaw > 2500 ? yRaw - 543 : yRaw);
          return { y, m: parseInt(dp[0]), d: parseInt(dp[1]), displayDate: dt[0], displayTime: dt[1]?.substring(0,5), moNo: cols[3], pCode: cols[7], pDesc: cols[8], qty: parseInt(cols[9]?.replace(/,/g,'')) || 0, color: cols[13], priceStr: (parseFloat(cols[14]?.replace(/,/g,'')) || 0).toLocaleString('th-TH',{minimumFractionDigits:2}) };
        }).filter(i => i.y === this.filters.year && i.m === this.filters.month && i.d >= ds && i.d <= de).sort((a,b) => a.d - b.d);
        this.lastUpdate = new Date().toLocaleString('th-TH');
      } catch (e) { console.error(e); } finally { this.yieldLoading = false; }
    },
    resetSearch() { this.sapInput = ""; this.processDisplay(); },
    selectSuggestion(i) { this.sapInput = i.sap; this.showSuggestions = false; this.processDisplay(); },
    handleBlur() { setTimeout(() => this.showSuggestions = false, 200); },
    handleResize() { if (this.totalChartInstance) this.totalChartInstance.resize(); }
  }
};
</script>

<style lang="scss" scoped>
/* CSS ดั้งเดิมของคุณทั้งหมด 100% */
.dashboard-wrapper { padding: 20px; background: #f8f7fa; min-height: 100vh; font-family: 'Public Sans', sans-serif; }
.section-title { font-size: 1.5rem; font-weight: 700; color: #5e5873; }
.section-underline { height: 4px; width: 50px; background: #7367f0; border-radius: 10px; }
.stats-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 24px; }
.stat-card { background: white; border-radius: 12px; border: 1px solid #ebe9f1; overflow: hidden; transition: transform 0.2s; &:hover { transform: translateY(-5px); } }
.card-indicator { height: 5px; width: 100%; background: #7367f0; &.green { background: #28c76f; } &.red { background: #ea5455; } }
.stat-body { padding: 20px; .stat-value { font-size: 1.6rem; font-weight: 700; margin: 8px 0; } }
.status-badge { display: inline-block; padding: 4px 10px; border-radius: 4px; font-size: 0.75rem; font-weight: bold; }
.bg-light-success { background: #e8f9ee; color: #28c76f; }
.bg-light-danger { background: #ffeeef; color: #ea5455; }
.custom-autocomplete { position: absolute; top: 100%; left: 0; right: 0; background: white; border: 1px solid #d8d6de; z-index: 1000; border-radius: 6px; max-height: 250px; overflow-y: auto; }
.autocomplete-item { padding: 10px; cursor: pointer; border-bottom: 1px solid #f8f8f8; &:hover { background: #f8f7fa; } }
.filter-container { border-radius: 12px; border: 1px solid #ebe9f1; }
.filter-label { font-size: 0.8rem; font-weight: 600; color: #82868b; }
.table-container { border-radius: 12px; border: 1px solid #ebe9f1; overflow: hidden; }

.table-custom { 
  table-layout: fixed; width: 100%;
  th { background: #f8f8f8; font-size: 0.8rem; color: #5e5873; padding: 12px; font-weight: 600; }
  td { padding: 12px; font-size: 0.85rem; color: #000 !important; border-bottom: 1px solid #f3f2f7; vertical-align: middle; }
  .desc-column { max-width: 250px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; cursor: help; }
}

.table-filter-input { height: 32px !important; font-size: 0.75rem; border-color: #d8d6de; }
.btn-retrieve { background: #7367f0 !important; border: none; font-weight: 600; }
@media (max-width: 992px) { .stats-grid { grid-template-columns: 1fr; } }
</style>