<script setup lang="ts">
import { requiredValidator } from "@/@core/utils/validators";
import { errorMessage, showActionResult, thousandSeparator } from "@/modules";
import { month_options, year_options } from "@/modules/options";
import ProcessButton from "@/page-components/ProcessButton.vue";
import axiosIns from "@/plugins/axios";
import { VForm } from "vuetify/components";

// INTERFACE
interface IEmits {
  (e: "invoiceAdded"): void;
}

// VARIABLE
const emits = defineEmits<IEmits>();
const is_on_process = ref(false);
const is_showing_modal = ref(false);
const invoice_form = ref<VForm>();
const invoice_data = ref({
  id_customer: null,
  month: null,
  year: null,
});
const options = ref({
  customer: [],
  month: month_options,
  year: year_options,
});

// FUNCTION
const getCustomerOptions = () => {
  axiosIns.get("options/customer").then((res) => {
    options.value.customer = res?.data?.customer_options || [];
  });
};
const saveInvoice = () => {
  invoice_form.value?.validate().then((form) => {
    if (form.valid) {
      is_on_process.value = true;
      axiosIns
        .post("invoice/add", {
          data: invoice_data.value,
        })
        .then(() => {
          showActionResult(undefined, undefined, "Invoice Telah Ditambahkan!");
          emits("invoiceAdded");
          invoice_form.value?.reset();
        })
        .catch((err) => {
          const message = errorMessage(err);
          showActionResult(true, "error", message);
        })
        .finally(() => {
          is_on_process.value = false;
          is_showing_modal.value = false;
        });
    }
  });
};
const customerAutocompleteFilter = (
  item_title: any,
  query_text: any,
  item: any
) => {
  const text_one = (item?.raw?.name ?? "").toLowerCase();
  const text_two = (item?.raw?.service_number ?? "").toString().toLowerCase();
  const search_text = (query_text ?? "").toLowerCase();
  return (
    text_one.indexOf(search_text) > -1 || text_two.indexOf(search_text) > -1
  );
};

// LIFECYCLE HOOKS
watch(is_showing_modal, () => {
  if (is_showing_modal.value) {
    getCustomerOptions();
  }
});
</script>
<template>
  <div>
    <div @click="is_showing_modal = true">
      <slot name="trigger-button">
        <VBtn size="40">
          <VIcon icon="tabler-plus" />
          <VTooltip activator="parent"> Tambah </VTooltip>
        </VBtn>
      </slot>
    </div>
    <VDialog :model-value="is_showing_modal" width="500" persistent>
      <DialogCloseBtn @click="is_showing_modal = false" />
      <VCard>
        <VCardItem>
          <template #prepend>
            <VIcon icon="tabler-plus" />
          </template>
          <template #title> Tambah Invoice </template>
        </VCardItem>
        <!-- EDIT FORM -->
        <VCardText>
          <VForm ref="invoice_form" @submit.prevent="saveInvoice()">
            <VRow>
              <!-- NAME -->
              <VCol cols="12">
                <VAutocomplete
                  v-model="invoice_data.id_customer"
                  :items="options.customer"
                  item-title="name"
                  item-value="_id"
                  clearable
                  :rules="[requiredValidator]"
                  :custom-filter="customerAutocompleteFilter"
                >
                  <template #label>
                    Nama Pelanggan <span class="text-error">*</span>
                  </template>
                  <template v-slot:item="{ props, item }">
                    <VListItem v-bind="props" class="px-2">
                      <template #subtitle>
                        <VChip
                          size="x-small"
                          variant="outlined"
                          color="primary"
                          class="font-weight-bold"
                        >
                          {{ item?.raw?.service_number }}
                        </VChip>
                      </template>
                    </VListItem>
                  </template>
                </VAutocomplete>
              </VCol>
              <!-- MONTH PERIOD -->
              <VCol cols="12" md="8" sm="12">
                <VAutocomplete
                  v-model="invoice_data.month"
                  :items="options.month"
                  :rules="[requiredValidator]"
                >
                  <template #label>
                    Bulan <span class="text-error">*</span>
                  </template>
                </VAutocomplete>
              </VCol>
              <!-- YEAR PERIOD -->
              <VCol cols="12" md="4" sm="12">
                <VAutocomplete
                  v-model="invoice_data.year"
                  :items="options.year"
                  :rules="[requiredValidator]"
                >
                  <template #label>
                    Tahun <span class="text-error">*</span>
                  </template>
                </VAutocomplete>
              </VCol>
              <!-- ACTION BUTTON -->
              <VCol cols="12">
                <VRow>
                  <VCol cols="6">
                    <VBtn
                      size="default"
                      block
                      color="error"
                      @click="is_showing_modal = false"
                    >
                      Batal
                    </VBtn>
                  </VCol>
                  <VCol cols="6">
                    <ProcessButton
                      :is_on_process="is_on_process"
                      size="default"
                      block
                      type="submit"
                      :disabled="is_on_process"
                    />
                  </VCol>
                </VRow>
              </VCol>
            </VRow>
          </VForm>
        </VCardText>
      </VCard>
    </VDialog>
  </div>
</template>
