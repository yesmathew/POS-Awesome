<template>
  <v-row justify="center">
    <v-dialog v-model="closingDialog" max-width="900px">
      <v-card>
        <v-card-title>
          <span class="headline primary--text">{{
            __('Closing POS Shift')
          }}</span>
        </v-card-title>
        <v-card-text class="pa-0">
          <v-container>
            <v-row>
              <v-col cols="12" class="pa-1">
                <template>
                  <v-data-table
                    :headers="headers"
                    :items="dialog_data.payment_reconciliation"
                    item-key="mode_of_payment"
                    class="elevation-1"
                    :items-per-page="itemsPerPage"
                    hide-default-footer
                  >
                    <template v-slot:item.closing_amount="props" >
                      <v-edit-dialog
                 
                      
                      :return-value.sync="props.item.closing_amount"
  >
        {{ formtCurrency(props.item.closing_amount) }}
        {{ currencySymbol(pos_profile.currency) }}
                        <template v-slot:input>
                          <v-text-field
                            v-model="props.item.closing_amount"
                            :rules="[max25chars]"
                            :label="frappe._('Edit')"
                            single-line
                            counter
                            :disabled="props.item.mode_of_payment === 'Total'"
                            type="number"
                            @change="addTotalRow"
                          ></v-text-field>
                        </template>
                      </v-edit-dialog>
        <!-- v-if="props.item.mode_of_payment!= 'Total'" -->

                    </template>

                    <template  v-slot:item.difference="{ item }">
                      {{ currencySymbol(pos_profile.currency) }}
                      {{
                        (item.difference = formtCurrency(
                          item.expected_amount - item.closing_amount
                        ))
                      }}</template
                    >
                    <template v-slot:item.opening_amount="{ item }">
                      {{ currencySymbol(pos_profile.currency) }}
                      {{ formtCurrency(item.opening_amount) }}</template
                    >
                    <template v-slot:item.expected_amount="{ item }">
                      {{ currencySymbol(pos_profile.currency) }}
                      {{ formtCurrency(item.expected_amount) }}</template
                    >

                    
                    
                    
                  </v-data-table>
                  
                </template>
              </v-col>
              </v-row>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="error" dark @click="close_dialog">{{
            __('Close')
          }}</v-btn>
          <v-btn color="success" dark @click="submit_dialog">{{
            __('Submit')
          }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import { evntBus } from '../../bus';
import format from '../../format';
export default {
  mixins: [format],
  data: () => ({
    closingDialog: false,
    itemsPerPage: 20,
    dialog_data: {},
    pos_profile: '',
    headers: [
      {
        text: __('Mode of Payment'),
        value: 'mode_of_payment',
        align: 'start',
        sortable: true,
      },
      {
        text: __('Opening Amount'),
        align: 'end',
        sortable: true,
        value: 'opening_amount',
      },
      {
        text: __('Closing Amount'),
        value: 'closing_amount',
        align: 'end',
        sortable: true,
      },
    ],
    max25chars: (v) => v.length <= 20 || 'Input too long!', // TODO : should validate as number
    pagination: {},
  }),
  watch: {},

  
  methods: {
    calculateTotals() {

      
      if (!this.dialog_data.payment_reconciliation) {
        return { totalOpening: 0, totalClosing: 0, totalExpected: 0, totalDifference: 0 };
      }

      let totalOpening = this.dialog_data.payment_reconciliation.reduce(
        (sum, item) => sum + (parseFloat(item.opening_amount) || 0),
        0
      );
      let totalClosing = this.dialog_data.payment_reconciliation.reduce(
        (sum, item) => sum + (parseFloat(item.closing_amount) || 0),
        0
        
      );
   
      
      let totalExpected = this.dialog_data.payment_reconciliation.reduce(
        (sum, item) => sum + (parseFloat(item.expected_amount) || 0),
        0
      );
      let totalDifference = totalExpected - totalClosing;

      return { totalOpening, totalClosing, totalExpected, totalDifference };
    },


    

  addTotalRow() {
  let totals = this.calculateTotals();

  // Remove the "Total" row and store other rows separately
  let filteredRows = this.dialog_data.payment_reconciliation.filter(
    (item) => item.mode_of_payment !== "Total"
  );

  // Append the "Total" row at the end
  filteredRows.push({
    idx: filteredRows.length + 1,  // Ensure a proper index
    mode_of_payment: "Total",
    opening_amount: totals.totalOpening,
    closing_amount: totals.totalClosing,
    expected_amount: totals.totalExpected,
    difference: totals.totalDifference.toFixed(2),
    parentfield: "payment_reconciliation",
    __islocal: 1
  });

  // Update the list while maintaining order
  this.dialog_data.payment_reconciliation = filteredRows;
},




 close_dialog() {  
      this.closingDialog = false;
    },
    async submit_dialog() {
  // Remove the "Total" row before submitting
  this.dialog_data.payment_reconciliation = this.dialog_data.payment_reconciliation.filter(
    (item) => item.mode_of_payment !== "Total"
  );



  evntBus.$emit('submit_closing_pos', this.dialog_data);  

  this.closingDialog = false;
}
  },

  created: function () {
    evntBus.$on('open_ClosingDialog', (data) => {
      this.closingDialog = true;
      this.dialog_data = data;
      if (this.dialog_data.payment_reconciliation.length) {
        this.addTotalRow();
      }
    });
     evntBus.$on('register_pos_profile', (data) => {
      this.pos_profile = data.pos_profile;
      // Reset headers to base structure first
      this.headers = [
        {
          text: __('Mode of Payment'),
          value: 'mode_of_payment',
          align: 'start',
          sortable: true,
        },
        {
          text: __('Opening Amount'),
          align: 'end',
          sortable: true,
          value: 'opening_amount',
        },
        {
          text: __('Closing Amount'),
          value: 'closing_amount',
          align: 'end',
          sortable: true,
        },
      ];

      // Conditionally add extra columns
      if (!this.pos_profile?.hide_expected_amount) {
        this.headers.push(
          {
            text: __('Expected Amount'),
            value: 'expected_amount',
            align: 'end',
            sortable: false,
          },
          {
            text: __('Difference'),
            value: 'difference',
            align: 'end',
            sortable: false,
          }
        );
      }
    });


    
  },
};

</script>
