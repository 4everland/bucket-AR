<template>
  <div>
    <div class="d-flex al-c mt-3">
      <img src="img/icon/nav-folder.svg" height="14" class="mr-2" />
      <v-breadcrumbs :items="navItems" class="pl-0">
        <template v-slot:divider>
          <v-icon size="20" color="#aaa">mdi-chevron-right</v-icon>
        </template>
      </v-breadcrumbs>
      <div class="d-flex ml-auto shrink-0">
        <nav-item icon="ic-sync" unit="MB">{{ usageInfo.arSyncing }}</nav-item>
        <nav-item icon="ic-synced" unit="MB" class="ml-7">{{
          usageInfo.arSynced
        }}</nav-item>
        <nav-item unit="Objects" class="ml-7">{{ total }}</nav-item>
      </div>
    </div>

    <!-- :show-select="list.length > 0" -->
    <v-data-table
      class="hide-bdb"
      :headers="headers"
      :items="list"
      :loading="tableLoading"
      v-model="selected"
      item-key="id"
      no-data-text=""
      loading-text=""
      checkbox-color="#0F8DFF"
      hide-default-footer
      disable-pagination
      @click:row="onRow"
    >
      <template v-slot:item.name="{ item }">
        <span>{{ item.name.cutStr(20, 10) }}</span>
        <e-tooltip right v-if="item.isDeleted">
          <v-icon slot="ref" color="#333" size="18" class="pa-1 d-ib ml-2"
            >mdi-alert-circle-outline</v-icon
          >
          <span>Deleted in Bucket</span>
        </e-tooltip>
      </template>
      <template v-slot:item.arweaveHash="{ item }">
        <v-btn
          color="primary"
          rounded
          x-small
          text
          target="_blank"
          v-if="item.arweaveHash"
          @click.stop="onStop"
          :href="$arHashPre + item.arweaveHash"
        >
          <span class="d-ib line-1" style="width: 160px">
            {{ item.arweaveHash }}
          </span>
        </v-btn>
        <v-btn
          v-if="item.arweaveHash"
          icon
          small
          @click.stop="onStop"
          v-clipboard="item.arweaveHash"
          @success="$toast('Copied to clipboard !')"
        >
          <!-- <v-icon size="14" color="primary">mdi-content-copy</v-icon> -->
          <img src="img/icon/copy1.svg" width="11" />
        </v-btn>
      </template>
      <template v-slot:item.arweaveStatus="{ item }">
        <sync-state :val="item.arweaveStatus"></sync-state>
      </template>
    </v-data-table>

    <div class="ta-c mt-8" v-if="!list.length">
      <img src="img/empty1.svg" width="80" />
      <div class="mt-3 gray fz-14">
        {{ tableLoading ? `Loading files...` : `No files found` }}
      </div>
    </div>

    <div
      v-if="!finished"
      class="pd-20 gray ta-c fz-16 mt-5"
      :class="{
        'hover-1': !loadingMore,
      }"
      @click="onLoadMore"
      v-intersect="onLoadMore"
    >
      <span v-if="list.length" v-show="!tableLoading">
        {{ loadingMore ? "Loading..." : "Load More" }}
      </span>
    </div>
  </div>
</template>

<script>
import { mapState } from "vuex";

export default {
  data() {
    return {
      headers: [
        { text: "Name", value: "name" },
        { text: "Size", value: "size" },
        { text: "AR Hash", value: "arweaveHash" },
        { text: "Last Modified", value: "updateAt" },
        { text: "AR Status", value: "arweaveStatus" },
      ],
      list: [],
      total: 0,
      tableLoading: false,
      selected: [],
      finished: false,
      loadingMore: false,
      cursor: 0,
    };
  },
  computed: {
    ...mapState({
      usageInfo: (s) => s.usageInfo,
    }),
    path() {
      return this.$route.path;
    },
    navItems() {
      return [
        {
          text: "AR History",
          to: "/arweave",
          exact: true,
        },
      ];
    },
  },
  watch: {
    path() {
      if (this.path == "/arweave") {
        this.getList();
      }
    },
  },
  mounted() {
    this.getList();
  },
  methods: {
    onStop() {},
    onLoadMore() {
      if (this.tableLoading) return;
      this.loadingMore = true;
      this.getList();
    },
    async getList() {
      try {
        this.tableLoading = true;
        if (this.loadingMore) {
          this.cursor = this.next;
        } else {
          this.finished = false;
          this.cursor = 0;
        }
        const { data } = await this.$http.get("/arweave/objects", {
          params: {
            cursor: this.cursor,
          },
        });
        this.next = Math.max(1, data.page.next);
        const list = data.list.map((it, i) => {
          it.id = it.id || it.cid + i;
          it.name = it.key.replace(/^.+\//, "");
          it.size = this.$utils.getFileSize(it.size);
          it.updateAt = new Date(it.lastModified * 1e3).format();
          return it;
        });
        if (this.loadingMore) {
          this.loadingMore = false;
          this.list = [...this.list, ...list];
        } else {
          this.list = list;
        }
        this.finished = !data.page.hasNext;
        this.total = data.page.total;
        // console.log(this.list);
      } catch (error) {
        console.log(error);
      }
      this.tableLoading = false;
    },
    onRow(it) {
      // console.log(it);
      if (it.isDeleted) return this.$toast("The file was deleted in Bucket.");
      const link = `/arweave/${it.bucket}/${it.key}`;
      this.$router.push(link);
    },
  },
};
</script>