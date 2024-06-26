<template>
  <!-- mtr Test -->
  <div class="mtr-test-section mb-4">
    <div class="jn-title2">
      <h2 id="MTRTest" :class="{ 'mobile-h2': isMobile }">📡 {{ $t('mtrtest.Title') }}</h2>

    </div>
    <div class="text-secondary">
      <p>{{ $t('mtrtest.Note') }}</p>
      <p v-if="!isMobile">{{ $t('mtrtest.Note2') }}</p>
    </div>
    <div class="row">
      <div class="col-12 mb-3">
        <div class="card jn-card" :class="{ 'dark-mode dark-mode-border': isDarkMode }">
          <div class="card-body">
            <!-- Dropdown for IP Selection -->
            <div class="row mt-3 mb-3 align-items-center justify-content-center">
              <div class="col-12 col-md-auto">
                <label for="mtrIP" class="col-form-label">{{ $t('mtrtest.Note3') }}</label>
              </div>
              <div class="col-12 col-md-auto mt-2 mt-md-0">
                <div class="input-group ">
                    <select id="mtrIP" aria-label="Select IP to MTR" class="form-select jn-ping-form-select" v-model="selectedIP"
                      :class="{ 'bg-dark text-light': isDarkMode }">
                      <option disabled value="">{{ $t('mtrtest.SelectIP') }}</option>
                      <option v-for="ip in allIPs" :key="ip" :value="ip">{{ ip }}</option>
                    </select>

                    <button class="btn btn-primary" @click="startmtrCheck"
                      :disabled="mtrCheckStatus === 'running' || selectedIP === ''">
                      <span
                        v-if="mtrCheckStatus === 'idle' || mtrCheckStatus === 'finished' || mtrCheckStatus === 'error'">{{
                          $t('mtrtest.Run') }}</span>
                      <span v-if="mtrCheckStatus === 'running'" class="spinner-grow spinner-grow-sm"
                        aria-hidden="true"></span>
                    </button>
                </div>
              </div>
            </div>

            <!-- Result Display -->
            <div id="mtrresult" v-if="mtrResults.length > 0">
              <div class="accordion" id="mtrResultsAccordion" :data-bs-theme="isDarkMode ? 'dark' : ''">
                <div class="accordion-item" v-for="(result, index) in mtrResults" :key="result.country">
                  <h2 class="accordion-header" :id="'heading' + index">
                    <button class="accordion-button" type="button" data-bs-toggle="collapse"
                      :data-bs-target="'#collapse' + index" :aria-expanded="index === 0 ? 'true' : 'false'"
                      :aria-controls="'collapse' + index" :class="{ collapsed: index !== 0 }">
                      <span :class="'jn-fl fi fi-' + result.country.toLowerCase()"></span>&nbsp;<strong>{{ result.city }}, {{
                        result.country }}</strong>
                      <span v-if="!isMobile">&nbsp;-&nbsp;{{ result.network }}&nbsp;</span>
                      <span v-if="!isMobile" class="badge rounded-pill"
                        :class="isDarkMode ? 'text-bg-warning' : 'text-bg-success'">AS{{ result.asn }}</span>
                    </button>
                  </h2>
                  <div :id="'collapse' + index" class="accordion-collapse collapse" :class="{ show: index === 0 }"
                    :aria-labelledby="'heading' + index">
                    <div class="accordion-body">
                      <div class="card card-body border-0 mt-3"
                        :class="[isDarkMode ? 'bg-black text-light' : 'bg-light']">
                        <pre>{{ result.rawOutput }}</pre>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <div id="mtrresult-error" v-if="mtrCheckStatus === 'error'">
              <div class="alert alert-info " :data-bs-theme="isDarkMode ? 'dark' : ''">{{ $t('mtrtest.Error') }}</div>
            </div>

          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch } from 'vue';
import { useStore } from 'vuex';

export default {
  name: 'MTRtest',

  // 引入 Store
  setup() {
    const store = useStore();
    const isDarkMode = computed(() => store.state.isDarkMode);
    const isMobile = computed(() => store.state.isMobile);
    let allIPs = computed(() => {
      const _allIPs = store.state.Global_ipDataCards;
      return _allIPs.filter(ip => ip && !ip.includes(' ') && !ip.includes(':'));
    });

    return {
      isDarkMode,
      isMobile,
      allIPs,
    };
  },

  data() {
    return {
      selectedIP: '',
      mtrResults: {},
      mtrCheckStatus: "idle",
    }
  },

  methods: {

    // 发起 mtr 测试
    startmtrCheck() {
      this.$trackEvent('Section', 'StartClick', 'MTRTest');
      // 清空上一次结果
      this.mtrResults = [];
      let tryCount = 0;
      // 子函数：发起 mtr 请求
      const sendmtrRequest = async () => {
        this.mtrCheckStatus = "running";
        try {
          const response = await fetch("https://api.globalping.io/v1/measurements", {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
            },
            body: JSON.stringify({
              limit: 16,
              locations: [
                { country: "HK" },
                { country: "TW" },
                { country: "CN" },
                { country: "JP" },
                { country: "SG" },
                { country: "IN" },
                { country: "RU" },
                { country: "US" },
                { country: "CA" },
                { country: "AU" },
                { country: "GB" },
                { country: "DE" },
                { country: "FR" },
                { country: "BR" },
                { country: "ZA" },
                { country: "SA" },
              ],
              target: this.selectedIP, // 使用用户选中的 IP 地址
              type: "mtr",
              measurementOptions: {
                "port": 80,
                "protocol": "ICMP"
              }
            })
          });

          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
          }

          return await response.json();
        } catch (error) {
          console.error("Error sending mtr request:", error);
        }
      };

      // 子函数：获取 mtr 结果
      const fetchmtrResults = async (id) => {
        try {
          const response = await fetch(`https://api.globalping.io/v1/measurements/${id}`);
          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
          }

          const data = await response.json();
          this.processmtrResults(data);

          if (data.status === "in-progress" && tryCount < 4) {
            setTimeout(() => fetchmtrResults(id), 1000);
            tryCount++;
          } else {
            // 如果 this.mtrResults 是空数组，返回错误信息
            if (this.mtrResults.length === 0) {
              this.mtrCheckStatus = "error";
            } else {
              this.mtrCheckStatus = "finished";
            }
          }
        } catch (error) {
          console.error("Error fetching mtr results:", error);
        }
      };

      // 执行流程
      sendmtrRequest().then(data => {
        if (data && data.id) {
          setTimeout(() => {
            fetchmtrResults(data.id);
          }, 1000);
        }
      });
    },
    processmtrResults(data) {
      const cleanedData = data.results
        .filter(item => item.result.status === "finished")
        .filter(item => item.result.rawOutput !== null)
        .map(item => ({
          country: item.probe.country,
          city: item.probe.city,
          network: item.probe.network,
          asn: item.probe.asn,
          rawOutput: item.result.rawOutput,
        }));

      this.mtrResults = cleanedData;
    },


  },
}
</script>

<style scoped>
.jn-focus-remove {
  outline: none;
}</style>
