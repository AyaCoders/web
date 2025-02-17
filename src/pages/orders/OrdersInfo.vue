<template>
  <div class="container mx-auto px-6 flex justify-center items-center py-10">
    <div v-if="isLoading" class="flex flex-col items-center justify-center gap-3">
      <md-linear-progress indeterminate />
      <span class="body-medium">Fetching order...</span>
    </div>

    <div v-else class="flex flex-col justify-center items-center w-full lg:w-3/4 xl:w-1/2">
      <div class="flex gap-5 flex-col md:flex-row justify-between w-full mb-3">
        <div>
          <h2 class="font-semibold title-large rounded-md w-full text-left">
            # {{ route.params.reference || order?.reference }}
          </h2>
          <h5 class="title-small w-full text-left">{{ order?.date_stamp ? getReadableDate(order?.date_stamp) : 'Invalid date' }}</h5>
        </div>
        <div>
          <md-outlined-select v-if="route.params.reference" v-model="status" label="Status" @change.prevent="onStatuChange" :disabled="isCompleted">
            <md-select-option
              v-for="option in statuses"
              :key="option.value"
              :value="option.value"
            >
              <span slot="headline">{{ option.label }}</span>
            </md-select-option>
          </md-outlined-select>
        </div>
      </div>

      <md-divider />

      <div class="student-info body-medium">
        <div>Name</div>
        <div>{{ order?.first_name }} {{ order?.last_name }}</div>
        <div>Email</div>
        <div>{{ order?.email_address }}</div>
        <div>Student ID</div>
        <div>{{ order?.student_id }}</div>
        <div>Course</div>
        <div>{{ order?.course == 0 ? 'BSCS' : order?.course && store.courses ? store.courses[order?.course] : "Unknown" }} {{ order?.year_level }}</div>
        <div>Remarks</div>
        <div>{{ order?.user_remarks || "Empty" }}</div>
        <div>Mode of Payment</div>
        <div v-if="order?.mode_of_payment === ModeOfPayment.WALK_IN">Walk-in</div>
        <div v-else>
          <a
            :href="getPhotoLink(order?.reference || 0, true)"
            class="border-b border-dashed border-outline"
            title="View submitted receipt"
            data-fancybox
          >
            GCash
          </a>
        </div>
        <div>Status</div>
        <div>{{ mapOrderStatusLabel(order?.status) }}</div>
        <div>Date Completed</div>
        <div class="text-outline">{{ order?.status === OrderStatus.COMPLETED || status === OrderStatus.COMPLETED ? (order?.status_updated ? getReadableDate(order.status_updated) : 'Invalid date') : '...' }}</div>
        <div>Quantity</div>
        <div>{{ order?.quantity }}</div>
      </div>

      <div class="flex justify-between mt-5 w-full bg-surface-container p-6 rounded-2xl text-on-surface-variant">
        <div class="flex flex-col justify-between">
          <div>
            <h3 class="text-base font-medium">
              {{ order?.product_name }}
            </h3>
            <h5 class="text-xs">{{ order?.variations_name || 'Standard' }}</h5>
          </div>

          <h3 class="body-medium">
            {{ toCurrency(order?.product_price || 0) }}
            
            <div class="text-outline inline-block ml-[1px]">
              &#215; {{ order?.quantity }}
            </div>
          </h3>
        </div>
        <div>
          <div class="h-24 flex items-center">
            <VImage
              v-if="order?.photos_hash || order?.variations_photo_hash"
              :src="getPhotoLink(order?.variations_photo_hash || order?.photos_hash)"
              :alt="order?.product_name"
              :w-full="false"
              h-full
            />
            <ImageTemplate v-else />
          </div>
        </div>
      </div>

      <div class="flex justify-end flex-col w-full mt-5 items-end gap-1">
        <span class="font-medium">Total</span>
        <span class="font-medium text-primary title-large">{{ toCurrency((order?.product_price || 0) * (order?.quantity || 0)) }}</span>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted } from 'vue';
import { useRoute } from 'vue-router';
import { toast } from 'vue3-toastify';
import { useStore, useDialog } from '~/store';
import { Endpoints, makeRequest } from '~/network/request';
import { getPhotoLink } from '~/utils/network';
import { Fancybox } from "@fancyapps/ui";
import { setPageTitle } from '~/utils/page';
import { getReadableDate } from '~/utils/date';
import { mapOrderStatusLabel } from '~/utils/page';
import { toCurrency } from '~/utils/string';
import { ModeOfPayment, OrderStatus } from '~/types/enums';
import { OrderEnum } from '~/types/models';

import "@material/web/menu/menu";
import "@material/web/menu/menu-item";
import "@material/web/divider/divider";
import "@material/web/progress/linear-progress";
import "@material/web/select/outlined-select";
import "@material/web/select/select-option";
import "@fancyapps/ui/dist/fancybox/fancybox.css";

import VImage from '~/components/VImage.vue';
import ImageTemplate from '~/composables/ImageTemplate.vue';

const route = useRoute();
const store = useStore();
const dialog = useDialog();
const isLoading = ref(true);
const order = ref<FullOrderModel>();
const isCompleted = ref(false);
const status = ref();
const currentStatus = ref();

const statuses = [
  { value: OrderStatus.PENDING_PAYMENT, label: "Pending" },
  { value: OrderStatus.COMPLETED, label: "Completed" },
  { value: OrderStatus.CANCELLED_BY_USER, label: "Cancelled by user" },
  { value: OrderStatus.CANCELLED_BY_ADMIN, label: "Cancelled by admin" },
  { value: OrderStatus.REMOVED, label: "Removed" },
  { value: OrderStatus.REJECTED, label: "Rejected" },
];

onMounted(() => {
  store.isLoading = true;

  // If using unique ID
  if (route.params.uniqueId) {
    // Fetch order
    makeRequest<FullOrderModel, { uniqueId: string }>("GET", Endpoints.OrdersUnique, {
      uniqueId: route.params.uniqueId as string
    }, processData);
    return;
  }

  // Fetch order
  makeRequest<FullOrderModel, { reference: string }>("GET", Endpoints.OrdersReference, {
    reference: route.params.reference as string
  }, processData);
});

function onStatuChange(ev: { target: { value: OrderStatus }}) {
  status.value = currentStatus.value;

  if (order.value?.id === undefined) {
    toast.error("ID is null");
  }

  if (ev.target.value === OrderStatus.COMPLETED) {
    const id = dialog.open({
      title: "Complete Order?",
      message: "It can't be changed once it has been marked as complete. Are you sure you want to continue?",
      ok: {
        text: "Yes, complete order",
        click() {
          dialog.close(id);
          updateStatus(order.value!.id, OrderStatus.COMPLETED);
        }
      },
      cancel: {
        text: "No, cancel",
        click() {
          dialog.close(id);
          status.value = currentStatus.value;
        }
      }
    });

    return;
  }

  if (order.value?.id && ev.target.value !== undefined) {
    updateStatus(order.value.id, ev.target.value);
    return;
  }
}

/**
 * Process order data
 */
function processData(response: ServerResponse<FullOrderModel>) {
  isLoading.value = false;
  store.isLoading = false;

  if (response.success) {
    order.value = response.data;
    status.value = order.value.status;
    currentStatus.value = order.value.status;

    if (order.value.status === OrderStatus.COMPLETED) {
      isCompleted.value = true;
    }

    // Set page title
    setPageTitle("Order #" + order.value.reference);
    // Bind fancybox
    setTimeout(() => {
      Fancybox.bind("[data-fancybox]", {
        Toolbar: {
          display: {
            left: ["infobar"],
            middle: [
              "zoomIn", "zoomOut", "toggle1to1", 
              "rotateCCW", "rotateCW", "flipX", "flipY",
            ],
            right: [
              "iterateZoom",
              "download",
              "fullscreen",
              "close"
            ]
          }
        }
      });
    }, 0);

    return;
  }

  toast.error(response.message);
}

/**
 * Update order status
 */
function updateStatus(orderId: string, newStatus: OrderStatus) {
  store.isLoading = true;

  makeRequest<FullOrderModel, { id: string, key: string, value: OrderStatus }>("PUT", Endpoints.OrdersKey, {
    id: orderId,
    key: OrderEnum.status,
    value: newStatus
  }, response => {
    store.isLoading = false;

    if (response.success) {
      currentStatus.value = newStatus;
      status.value = newStatus;

      if (newStatus === OrderStatus.COMPLETED) {
        isCompleted.value = true;

        if (order.value) {
          order.value = response.data;
        }
      }

      toast.success(response.message);
      return;
    }

    toast.error(response.message);
  });
}
</script>

<style lang="scss" scoped>
.student-info {
  @apply grid grid-cols-2 w-full mt-5 gap-2;

  & > div:nth-child(even) {
    @apply text-right break-words;
  }
}
</style>