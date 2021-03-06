<template>
  <v-card class="table">
    <v-data-table
      :headers="headers"
      :custom-filter="customFilter"
      :items="suggestions"
      :search="search"
      multi-sort
      :mobile-breakpoint="0"
      :items-per-page="15"
      :footer-props="{ itemsPerPageOptions: [15, 30, 50, 100] }"
    >
      <template v-slot:top>
        <v-card-title>
          <v-text-field
            v-model="search"
            append-icon="search"
            label="Search"
            class="mx-2"
          />
        </v-card-title>
      </template>

      <template v-slot:item.title="{ item }">
        <p
          v-html="convertMarkDown(item.question.title, null)"
          @click="showSuggestionDialog(item)"
      /></template>

      <template v-slot:item.content="{ item }">
        <p
          v-html="convertMarkDown(item.question.content, null)"
          @click="showSuggestionDialog(item)"
      /></template>

      <template v-slot:item.status="{ item }">
        <v-chip :color="getStatusColor(item.status)" small>
          <span>{{ item.status }}</span>
        </v-chip>
      </template>

      <template v-slot:item.action="{ item }">
        <v-tooltip bottom>
          <template v-slot:activator="{ on }">
            <v-icon
              small
              class="mr-2"
              v-on="on"
              @click="showSuggestionDialog(item)"
              >visibility</v-icon
            >
          </template>
          <span>Show Suggestion</span>
        </v-tooltip>
        <v-tooltip bottom v-if="!item.approved">
          <template v-slot:activator="{ on }">
            <v-icon
              small
              class="mr-2"
              v-on="on"
              @click="addSuggestionReview(item)"
              data-cy="createSuggestionReviewButton"
              >add</v-icon
            >
          </template>
          <span>Add Suggestion Review</span>
        </v-tooltip>
      </template>
    </v-data-table>
    <add-suggestion-review-dialog
      v-if="currentSuggestionReview"
      v-model="addSuggestionReviewDialog"
      :suggestion="currentSuggestion"
      :suggestionReview="currentSuggestionReview"
      v-on:save-suggestion-review="onSaveSuggestionReview"
    />
    <show-suggestion-dialog
      v-if="currentSuggestion"
      :dialog="suggestionDialog"
      :suggestion="currentSuggestion"
      v-on:close-show-suggestion-dialog="onCloseShowSuggestionDialog"
    />
  </v-card>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import RemoteServices from '@/services/RemoteServices';
import { convertMarkDown } from '@/services/ConvertMarkdownService';
import Image from '@/models/management/Image';
import Suggestion from '@/models/management/Suggestion';
import SuggestionReview from '@/models/management/SuggestionReview';
import ShowSuggestionDialog from '@/views/student/suggestions/ShowSuggestionDialog.vue';
import AddSuggestionReviewDialog from '@/views/teacher/suggestions/AddSuggestionReviewDialog.vue';

@Component({
  components: {
    'show-suggestion-dialog': ShowSuggestionDialog,
    'add-suggestion-review-dialog': AddSuggestionReviewDialog
  }
})
export default class TeacherSuggestionsView extends Vue {
  suggestions: Suggestion[] = [];
  currentSuggestion: Suggestion | null = null;
  currentSuggestionReview: SuggestionReview | null = null;
  suggestionDialog: boolean = false;
  addSuggestionReviewDialog: boolean = false;
  search: string = '';

  headers: object = [
    { text: 'Title', value: 'title', align: 'center' },
    { text: 'Question', value: 'content', align: 'left' },
    { text: 'Status', value: 'status', align: 'center' },
    {
      text: 'Creation Date',
      value: 'creationDate',
      align: 'center'
    },
    {
      text: 'Image',
      value: 'image',
      align: 'center',
      sortable: false
    },
    {
      text: 'Actions',
      value: 'action',
      align: 'center',
      sortable: false
    }
  ];

  async created() {
    await this.$store.dispatch('loading');
    try {
      this.suggestions = await RemoteServices.getTeacherSuggestions();
    } catch (error) {
      await this.$store.dispatch('error', error);
    }
    await this.$store.dispatch('clearLoading');
  }

  customFilter(value: string, search: string, suggestion: Suggestion) {
    // noinspection SuspiciousTypeOfGuard,SuspiciousTypeOfGuard
    return (
      search != null &&
      JSON.stringify(suggestion)
        .toLowerCase()
        .indexOf(search.toLowerCase()) !== -1
    );
  }

  getStatusColor(status: string) {
    if (status === 'REJECTED') return 'red';
    else if (status === 'PENDING') return 'orange';
    else return 'green';
  }

  convertMarkDown(text: string, image: Image | null = null): string {
    return convertMarkDown(text, image);
  }

  showSuggestionDialog(suggestion: Suggestion) {
    this.currentSuggestion = suggestion;
    this.suggestionDialog = true;
  }

  onCloseShowSuggestionDialog() {
    this.suggestionDialog = false;
  }

  addSuggestionReview(suggestion: Suggestion) {
    this.currentSuggestion = suggestion;
    this.$store.dispatch('currentSuggestion', this.currentSuggestion);
    this.currentSuggestionReview = new SuggestionReview();
    this.addSuggestionReviewDialog = true;
  }

  async onSaveSuggestionReview() {
    this.addSuggestionReviewDialog = false;
    this.currentSuggestion = null;
    this.currentSuggestionReview = null;
  }
}
</script>

<style lang="scss" scoped>
.question-textarea {
  text-align: left;

  .CodeMirror,
  .CodeMirror-scroll {
    min-height: 200px !important;
  }
}
.option-textarea {
  text-align: left;

  .CodeMirror,
  .CodeMirror-scroll {
    min-height: 100px !important;
  }
}
</style>
