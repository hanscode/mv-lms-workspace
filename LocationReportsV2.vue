<template>
  <div class="bg-white px-4 py-4 rounded shadow border border-vento-gray-200">
    <!-- Show loading state while user is being fetched -->
    <div v-if="isLoading" class="flex justify-center py-8">
      <span class="text-gray-500">Loading...</span>
    </div>

    <!-- Using Table component with custom toolbar layout -->
    <div v-else-if="apiUrl && !isLoading">
      <Table
        ref="tableRef"
        :key="apiUrl"
        :columns="columns"
        :url="apiUrl"
        :filters="filters"
        :searchable="false"
        :show-total="true"
        :row_actions="rowActions"
        mode="api"
      >
        <!-- Custom search bar in left toolbar (matches original layout) -->
        <template #toolbarLeft>
          <BaseTextInput
            type="search"
            id="search registration"
            :model-value="searchTerm"
            placeholder="Search by keyword"
            @update:model-value="handleSearch"
            class="md:min-w-[300px]"
          />
        </template>

        <!-- Last update and extra buttons in right toolbar -->
        <template #toolbarRight>
          <div class="flex items-center gap-x-4">
            <!-- Extra buttons for district view -->
            <div v-if="extraButtons.length > 0" class="flex items-center gap-2">
              <NuxtLink v-for="(button, index) in extraButtons" :key="index" :to="button.href">
                <BaseButton appearance="outline" class="text-sm">
                  {{ button.label }}
                </BaseButton>
              </NuxtLink>
            </div>
            <!-- Last update timestamp -->
            <BaseText size="sm" class="mt-4 md:mt-0">Last update: {{ lastRunAtFormatted }}</BaseText>
          </div>
        </template>
      </Table>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue';
import Table from '~/components/ui/Table.vue';
import BaseButton from '~/components/ui/BaseButton.vue';
import BaseTextInput from '~/components/ui/BaseTextInput.vue';
import BaseText from '~/components/ui/BaseText.vue';
import type { Option as FormOption } from '~/types/form';
import type { TableColumn, TableRowAction } from '~/types/table';
import { mdiEye, mdiAccountTie, mdiAccountGroup } from '@mdi/js';
import PercentageDisplay from '~/components/Participant/Reports/PercentageDisplay.vue';
// Use the recommended Sanctum method for getting current user
import { useSanctumUser } from '#imports';

interface User {
  id: number;
  type: string;
  state?: { id: number };
  states?: Array<{ id: number }>;
  [key: string]: any;
}

interface ApiResponse {
  data: any[];
  meta?: any;
  last_run_at?: string;
  [key: string]: any;
}

interface Props {
  type: 'state' | 'region' | 'district';
  selectedSchoolYear: FormOption;
}

const props = defineProps<Props>();
const route = useRoute();
const tableRef = ref();
const isLoading = ref(true);
const lastRunAt = ref<string>('');
const searchTerm = ref<string>('');

// Use Sanctum's recommended method for getting current user
const user = useSanctumUser() as Ref<User | null>;

// Compute the API endpoint based on report type
const apiUrl = computed(() => {
  // Get the appropriate ID based on report type
  let id: string | number | undefined;

  if (props.type === 'state') {
    // For admin users testing, check if state ID is in route params first
    if (route.params.id) {
      id = Array.isArray(route.params.id) ? route.params.id[0] : route.params.id;
    }
    // For state reports, check if user has states array (multiple states)
    else if (user.value?.states && user.value.states.length > 0) {
      id = user.value.states[0].id;
    }
    // Or check if user has a single state
    else if (user.value?.state && user.value.state.id && user.value.state.id !== -1) {
      id = user.value.state.id;
    }
    // For admin users, default to state ID 1 for testing
    else if (user.value?.type === 'admin') {
      id = 1; // Default state ID for admin testing - Alabama
    }
    // If no state found, return empty
    if (!id) {
      return '';
    }
  } else {
    // For region and district, use route params
    id = Array.isArray(route.params.id) ? route.params.id[0] : route.params.id;
  }

  // Return empty if no valid ID
  if (!id) {
    return '';
  }

  // Build URL with school year if available
  let url = `/api/v1/reports/${props.type}s/${id}`;

  // Add school year as a query param in the base URL
  if (props.selectedSchoolYear?.value) {
    url += `?school_year_id=${props.selectedSchoolYear.value}`;
  }

  return url;
});

// Format the last update timestamp like the original
const lastRunAtFormatted = computed(() => {
  if (!lastRunAt.value) return '';
  return new Date(lastRunAt.value).toLocaleDateString('en-US', {
    month: 'long',
    day: 'numeric',
    year: 'numeric',
  });
});

// Extra buttons for district view
const extraButtons = computed(() => {
  if (props.type === 'district') {
    const id = route.params.id as string;
    return [
      {
        label: 'View district liaisons',
        href: `/reports/district/${id}/liaison`,
      },
      {
        label: 'View district essential staff',
        href: `/reports/district/${id}/es`,
      },
    ];
  }
  return [];
});

// Column definitions for the table
const columns = computed((): TableColumn[] => {
  const firstColumnName = props.type === 'state' ? 'Region' : props.type === 'region' ? 'District' : 'School';

  return [
    {
      name: firstColumnName,
      key: props.type === 'state' ? 'region_name' : props.type === 'region' ? 'district_name' : 'school_name',
      type: 'text' as const,
      sticky: 'left' as const,
    },
    {
      name: 'Liaisons',
      key: 'total_liaisons',
      type: 'text' as const,
    },
    {
      name: 'Liaison Completion',
      key: 'liaisons_completion_pct',
      type: 'custom' as const,
      component: PercentageDisplay,
    },
    {
      name: 'Essential Staff',
      key: 'total_essential_staff',
      type: 'text' as const,
    },
    {
      name: 'Staff Completion',
      key: 'essential_staff_completion_pct',
      type: 'custom' as const,
      component: PercentageDisplay,
    },
    {
      name: 'Overall Completion',
      key: 'overall_completion_pct',
      type: 'custom' as const,
      component: PercentageDisplay,
    },
  ];
});

// Filters - currently none for reports, but structure is ready for future use
const filters = ref([]);

// Row actions based on report type
const rowActions = computed((): TableRowAction[] => {
  const actions: TableRowAction[] = [];

  // View location action for state and region
  if (props.type === 'state' || props.type === 'region') {
    const actionLabel = props.type === 'state' ? 'View Region' : 'View District';
    actions.push({
      label: actionLabel,
      type: 'redirect' as const,
      icon: mdiEye,
      urlResolver: (row: any) => {
        const id = props.type === 'state' ? row.region_id : row.district_id;
        return props.type === 'state' ? `/reports/region/${id}` : `/reports/district/${id}`;
      },
    });
  }

  // View liaison and staff actions (not for district level)
  if (props.type !== 'district') {
    actions.push(
      {
        label: 'View Liaisons',
        type: 'redirect' as const,
        icon: mdiAccountTie,
        urlResolver: (row: any) => {
          // For state level: go to region liaisons
          // For region level: go to district liaisons
          if (props.type === 'state') {
            const regionId = row.region_id || row.id;
            return `/reports/region/${regionId}/liaison`;
          } else if (props.type === 'region') {
            const districtId = row.district_id || row.id;
            return `/reports/district/${districtId}/liaison`;
          }
          return '';
        },
      },
      {
        label: 'View Staff',
        type: 'redirect' as const,
        icon: mdiAccountGroup,
        urlResolver: (row: any) => {
          // For state level: go to region staff
          // For region level: go to district staff
          if (props.type === 'state') {
            const regionId = row.region_id || row.id;
            return `/reports/region/${regionId}/es`;
          } else if (props.type === 'region') {
            const districtId = row.district_id || row.id;
            return `/reports/district/${districtId}/es`;
          }
          return '';
        },
      }
    );
  }

  return actions;
});

// Fetch last run at data from API
const fetchLastRunAt = async () => {
  if (!apiUrl.value) return;

  try {
    const { data } = await useSanctumFetch<ApiResponse>(apiUrl.value);
    if (data.value?.last_run_at) {
      lastRunAt.value = data.value.last_run_at;
    }
  } catch (error) {
    console.warn('Failed to fetch last_run_at data:', error);
  }
};

// Initialize component
onMounted(async () => {
  // Wait a tick for user data to be available from Sanctum
  await nextTick();

  // Set loading to false after user is available
  isLoading.value = false;

  // Fetch last run at data
  await fetchLastRunAt();
});

// Handle search input - manually trigger Table component search
const handleSearch = (value: string) => {
  searchTerm.value = value;

  // Access the Table component's internal search functionality
  if (tableRef.value) {
    // Update the Table component's internal search term
    tableRef.value.searchedTerm = value;
    // Trigger the search by calling the component's search method
    if (tableRef.value.fetchDataFromApi) {
      tableRef.value.fetchDataFromApi();
    }
  }
};

// Watch for school year changes and refresh table
watch(
  () => props.selectedSchoolYear,
  () => {
    if (tableRef.value && apiUrl.value) {
      tableRef.value.fetchDataFromApi();
      // Also refetch last run at data
      fetchLastRunAt();
    }
  }
);

// Watch for apiUrl changes and fetch last run at data
watch(
  apiUrl,
  () => {
    if (apiUrl.value) {
      fetchLastRunAt();
    }
  }
);
</script>
