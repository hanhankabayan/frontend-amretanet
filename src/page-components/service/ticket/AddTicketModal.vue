<script setup lang="ts">
import { requiredValidator } from "@/@core/utils/validators";
import { errorMessage, roleFormatter, showActionResult } from "@/modules";
import ProcessButton from "@/page-components/ProcessButton.vue";
import axiosIns from "@/plugins/axios";
import { stateManagement } from "@/store";
import { VForm } from "vuetify/components";

// INTERFACE
interface IProps {
  user_options: any[];
}
interface IEmits {
  (e: "ticketAdded"): void;
}

// VARIABLE
const store = stateManagement();
const props = defineProps<IProps>();
const emits = defineEmits<IEmits>();
const is_on_process = ref(false);
const is_showing_modal = ref(false);
const options = ref({
  user: props?.user_options || [],
  odp: [],
  odc: [],
  title: [],
});
const ticket_form = ref<VForm>();
const ticket_data = ref({
  title: null,
  type: null,
  id_reporter: store.isCustomer ? store.getUser._id : null,
  id_assignee: null,
  id_odc: null,
  id_odp: null,
  description: null,
});

// FUNCTION
const getTicketTitleOptions = () => {
  axiosIns.get("options/ticket-title").then((res) => {
    options.value.title = res.data.ticket_title_options || [];
  });
};
const getODCOptions = () => {
  axiosIns.get("options/odc").then((res) => {
    options.value.odc = res.data.odc_options || [];
  });
};
const getODPOptions = () => {
  axiosIns.get("options/odp").then((res) => {
    options.value.odp = res.data.odp_options || [];
  });
};
const saveTicket = () => {
  ticket_form.value?.validate().then(({ valid: is_valid }) => {
    if (is_valid) {
      is_on_process.value = true;
      axiosIns
        .post("ticket/add", {
          data: ticket_data.value,
        })
        .then(() => {
          emits("ticketAdded");
          resetForm();
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
const resetForm = () => {
  ticket_form.value?.reset();
};
const setTicketType = () => {
  const selected_title: any = options.value.title.find(
    (el: any) => el.title === ticket_data.value.title
  );
  if (selected_title) {
    ticket_data.value.type = selected_title.type;
    if (ticket_data.value.type !== "TKT") {
      ticket_data.value.id_reporter = null;
    }
  }
};
const customerAutocompleteFilter = (
  item_title: any,
  query_text: any,
  item: any
) => {
  const text_one = (item?.raw?.title ?? "").toString().toLowerCase();
  const text_two = (item?.raw?.service_number ?? "").toString().toLowerCase();
  const search_text = (query_text ?? "").toString().toLowerCase();

  return (
    text_one.indexOf(search_text) > -1 || text_two.indexOf(search_text) > -1
  );
};

// LIFECYCLE HOOKS
watch(is_showing_modal, () => {
  if (is_showing_modal.value) {
    getTicketTitleOptions();
    getODCOptions();
    getODPOptions();
  }
});
watch(props, () => {
  options.value.user = props.user_options;
});
</script>
<template>
  <div>
    <div @click="is_showing_modal = true">
      <slot name="trigger-button">
        <VBtn size="40" prepend-icon="tabler-plus">
          <VTooltip activator="parent"> Tambah Tiket </VTooltip>
        </VBtn>
      </slot>
    </div>
    <VDialog :model-value="is_showing_modal" max-width="500" persistent>
      <DialogCloseBtn @click="is_showing_modal = false" />
      <VCard>
        <VCardItem>
          <template #prepend>
            <VIcon icon="tabler-plus" />
          </template>
          <template #title> Tambah Tiket </template>
        </VCardItem>
        <VCardText>
          <VForm ref="ticket_form" @submit.prevent="saveTicket">
            <VRow>
              <!-- TITLE -->
              <VCol cols="12">
                <VAutocomplete
                  v-model="ticket_data.title"
                  :items="
                    store.isCustomer
                      ? options.title.filter((el:any) => el.type === 'TKT')
                      : options.title
                  "
                  :rules="[requiredValidator]"
                  clearable
                  @update:model-value="setTicketType"
                >
                  <template #label>
                    Judul Pesan <span class="text-error">*</span>
                  </template>
                  <template v-slot:item="{ props, item }">
                    <VListItem v-bind="props" class="px-2">
                      <template #title>
                        <span class="fs-14">
                          {{ item?.raw?.title }}
                        </span>
                      </template>
                      <template #subtitle>
                        <div class="d-flex gap-1">
                          <VChip
                            size="x-small"
                            variant="outlined"
                            class="font-weight-bold"
                          >
                            #{{ item.raw.type }}
                          </VChip>
                        </div>
                      </template>
                    </VListItem>
                  </template>
                </VAutocomplete>
              </VCol>
              <!-- REPORTER -->
              <VCol
                v-if="
                  (ticket_data.type === 'TKT' || ticket_data.type === 'PSB') &&
                  !store.isCustomer
                "
                cols="12"
              >
                <VAutocomplete
                  v-model="ticket_data.id_reporter"
                  :items="options.user.filter((el:any)=>el.role===99)"
                  :rules="[requiredValidator]"
                  clearable
                  :custom-filter="customerAutocompleteFilter"
                >
                  <template #label>
                    Pelanggan <span class="text-error">*</span>
                  </template>
                  <template v-slot:item="{ props, item }">
                    <VListItem v-bind="props" class="px-2">
                      <template #title>
                        <span class="fs-14">
                          {{ item?.raw?.title }}
                        </span>
                      </template>
                      <template #subtitle>
                        <div class="d-flex gap-1">
                          <VChip
                            v-if="item?.raw?.service_number"
                            size="x-small"
                            variant="outlined"
                          >
                            {{ item?.raw?.service_number }}
                          </VChip>
                          <VChip
                            size="x-small"
                            variant="outlined"
                            :color="roleFormatter(item?.raw?.role).color"
                          >
                            {{ roleFormatter(item?.raw?.role).title }}
                          </VChip>
                        </div>
                      </template>
                    </VListItem>
                  </template>
                </VAutocomplete>
              </VCol>
              <!-- ENGINEER -->
              <VCol v-if="!store.isCustomer" cols="12">
                <VAutocomplete
                  v-model="ticket_data.id_assignee"
                  :items="options.user.filter((el:any)=>el.role ===5)"
                  :rules="[requiredValidator]"
                  clearable
                >
                  <template #label>
                    Teknisi <span class="text-error">*</span>
                  </template>
                  <template v-slot:item="{ props, item }">
                    <VListItem v-bind="props" class="px-2">
                      <template #title>
                        <span class="fs-14">
                          {{ item?.raw?.title }}
                        </span>
                      </template>
                      <template #subtitle>
                        <div class="d-flex gap-1">
                          <VChip
                            size="x-small"
                            variant="outlined"
                            :color="roleFormatter(item?.raw?.role).color"
                          >
                            {{ roleFormatter(item?.raw?.role).title }}
                          </VChip>
                        </div>
                      </template>
                    </VListItem>
                  </template>
                </VAutocomplete>
              </VCol>
              <!-- ODC -->
              <VCol v-if="!store.isCustomer" cols="12">
                <VAutocomplete
                  v-model="ticket_data.id_odc"
                  label="ODC"
                  :items="options.odc"
                  clearable
                />
              </VCol>
              <!-- ODP -->
              <VCol v-if="!store.isCustomer" cols="12">
                <VAutocomplete
                  v-model="ticket_data.id_odp"
                  label="ODP"
                  :items="options.odp"
                  clearable
                />
              </VCol>
              <!-- DESCRIPTION -->
              <VCol cols="12">
                <VTextarea
                  v-model="ticket_data.description"
                  label="Isi Pesan"
                  :rules="[requiredValidator]"
                >
                  <template #label>
                    Deskripsi Pesan <span class="text-error">*</span>
                  </template>
                </VTextarea>
              </VCol>
              <!-- ACTION BUTTON -->
              <VCol cols="12">
                <VRow>
                  <VCol cols="6">
                    <VBtn
                      size="default"
                      block
                      color="error"
                      @click="(is_showing_modal = false), resetForm()"
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
