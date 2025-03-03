<template>
	<div class="h-full">
		<slot name="body" :manager="manager" />
	</div>
</template>

<script>
import {
	ref,
	computed,
	watch,
	provide,
	nextTick,
	onMounted,
} from "vue";
import { useRouter, useRoute } from "vue-router";
import { createListResource, createResource } from "frappe-ui";
import { useAuthStore } from "@/stores/auth";

export default {
	name: "ListManager",
	props: ["options"],
	setup(props, context) {
		const router = useRouter();
		const route = useRoute();
		const authStore = useAuthStore();

		const options = ref({
			handleRowClick: () => {},
			urlQueryFilters: props.options.urlQueryFilters || false,
			saveFiltersLocally: props.options.saveFiltersLocally || false,
			cache: props.options.cache || null,
			fields: [...new Set([...(props.options.fields || []), "name"])],
			doctype: props.options.doctype,
			filters: props.options.filters || [],
			pageLength: props.options.limit || 20,
			start: 0,
			orderBy: props.options.order_by || "",
		});

		const listResource = createListResource(
			{
				type: "list",
				url: "helpdesk.extends.client.get_list",
				cache: false,
				doctype: options.value.doctype,
				fields: options.value.fields,
				orderBy: options.value.orderBy,
				filters: options.value.filters,
				pageLength: options.value.pageLength,
				start: options.value.start,
				realtime: true,
				pageLength: 3,
			},
			context
		);
		const list = computed(() => {
			countResource.fetch({
				doctype: options.value.doctype,
				filters: options.value.filters,
			});
			return listResource.list.data || [];
		});
		const loading = computed(() => {
			return listResource.list.loading;
		});

		const countResource = createResource(
			{
				url: "frappe.client.get_count",
			},
			context
		);
		const totalCount = computed(() => {
			return countResource.data || 0;
		});

		const reload = () => {
			selectedItems.value = {};
			listResource.reload();
		};
		const fetch = () => {
			listResource.list.fetch();
		};
		const update = (newOptions = null) => {
			if (newOptions) {
				if (newOptions.filters) options.value.filters = newOptions.filters;
				if (newOptions.orderBy) options.value.orderBy = newOptions.orderBy;
			}

			listResource.update({
				...options.value,
			});
		};

		const getPage = (page) => {
			if (page < 0 || page > totalPages.value) return;
			options.value.start = (page - 1) * options.value.pageLength;
			update();
			fetch();
		};
		const nextPage = () => {
			if (currentPageNumber.value == totalPages.value) return;
			options.value.start += options.value.pageLength;
			update();
			fetch();
		};
		const previousPage = () => {
			if (currentPageNumber.value == 1) return;
			let newStart = options.value.start - options.value.pageLength;
			if (newStart < 0) newStart = 0;
			options.value.start = newStart;
			update();
			fetch();
		};
		const hasNextPage = computed(() => {
			return options.value.start + options.value.pageLength < totalCount.value;
		});
		const hasPreviousPage = computed(() => {
			return options.value.start > 0;
		});
		const currentPageNumber = computed(() => {
			return Math.ceil(options.value.start / options.value.pageLength) + 1;
		});
		const totalPages = computed(() => {
			return Math.ceil(totalCount.value / options.value.pageLength);
		});

		const sudoFilters = ref([]);
		const generateExecutableFilter = (filter) => {
			let mapOperator = (x) => {
				switch (x) {
					case "is":
						return "=";
					case "is not":
						return "!=";
					case "before":
						return "<";
					case "after":
						return ">";
					default:
						return x;
				}
			};
			let mapValue = (x, type) => {
				switch (type) {
					case "like":
						return `%${x}%`;
					case "not like":
						return `%${x}%`;
					default:
						return x;
				}
			};

			let filterType = filter.filter_type;
			if (filter.fieldname == "_assign") {
				filterType = filter.filter_type == "is" ? "like" : "not like";
			}

			let executableFilter = [
				filter.fieldname,
				mapOperator(filterType),
				mapValue(
					filter.fieldname == "_assign" && filter.value == "@me"
						? authStore.userId
						: filter.value,
					filterType
				),
			];
			return executableFilter;
		};
		const addFilters = (sudoFilters, addToUrlQuery) => {
			if (addToUrlQuery) {
				let query = {};
				sudoFilters.forEach((filter) => {
					let fieldname = filter.fieldname;
					let filter_type = filter.filter_type;
					let value =
						filter.fieldname == "_assign" && filter.value == "@me"
							? authStore.userId
							: filter.value;

					query[fieldname] = JSON.stringify([filter_type, value]);
				});
				router.replace({ query });
				if (
					Object.keys(query).length == 0 &&
					options.value.saveFiltersLocally
				) {
					applyFilters();
				}
			} else {
				let executableFilters = [];
				for (let i in sudoFilters) {
					executableFilters.push(generateExecutableFilter(sudoFilters[i]));
				}
				applyFilters(sudoFilters, executableFilters);
			}
		};
		const applyFilters = (sudoFilters = [], executableFilters = []) => {
			if (options.value.saveFiltersLocally) {
				localStorage.setItem(
					`list_filters_${route.path}`,
					JSON.stringify(sudoFilters || [])
				);
			}
			manager.value.update({
				filters: executableFilters,
			});
		};

		const toggleOrderBy = (field) => {
			let newOrderBy = `${field} desc`;
			const oldOrderBy = options.value?.orderBy;
			if (oldOrderBy) {
				if (oldOrderBy.split(" ")[0] === newOrderBy.split(" ")[0]) {
					newOrderBy = `${field} ${
						oldOrderBy.split(" ")[1] === "desc" ? "asc" : "desc"
					}`;
				}
			}
			update({ orderBy: newOrderBy });
			reload();
		};

		const onClick = (rowData) => {
			if (selectionMode.value == 1) {
				selectionMode.value = 2;
			} else if (selectionMode.value == 2) {
				manager.value.select(rowData);
			} else {
				options.value.handleRowClick(rowData);
			}
		};

		const select = (rowData) => {
			if (selectionMode.value == 0) {
				selectionMode.value = 1;
			}
			if (rowData.name in selectedItems.value) {
				delete selectedItems.value[rowData.name];
				if (Object.keys(selectedItems.value).length == 0) {
					selectionMode.value = 0;
				}
			} else {
				selectedItems.value[rowData.name] = rowData;
			}
		};
		const unselect = () => {
			selectedItems.value = {};
		};
		const selectAll = () => {
			if (allItemsSelected.value) {
				manager.value.unselect();
			} else {
				for (let i = 0; i < manager.value.list.length; i++) {
					selectedItems.value[manager.value.list[i].name] =
						manager.value.list[i];
				}
			}
			context.emit("selection", selectedItems.value);
		};

		const selectionMode = ref(0);
		const selectedItems = ref({});
		watch(selectedItems.value, (newValue) => {
			context.emit("selection", newValue);
		});
		const allItemsSelected = computed(() => {
			if (manager.value.loading) {
				return false;
			} else {
				if (manager.value.list.length > 0) {
					return (
						Object.keys(selectedItems.value).length == manager.value.list.length
					);
				}
				return false;
			}
		});
		const itemSelected = (rowData) => {
			return rowData.name in selectedItems.value;
		};

		const convertFieldNameToLabel = (fieldname) => {
			switch (fieldname) {
				case "_assign":
					return "Assigned to";
				case "_seen":
					return false;
				default:
					fieldname = fieldname.replace(/_/g, " ");
					return fieldname.charAt(0).toUpperCase() + fieldname.slice(1);
			}
		};
		const convertUrlQueryToFilters = () => {
			let urlQuery = route.query;
			let filters = [];

			for (let key in urlQuery) {
				let filterType = "like";
				let value = "";
				urlQuery[key] = JSON.parse(urlQuery[key]);
				if (urlQuery[key].constructor.name == "Array") {
					if (urlQuery[key].length == 1) {
						value = urlQuery[key][0];
					} else {
						filterType = urlQuery[key][0];
						value = urlQuery[key][1];
					}
				} else {
					value = urlQuery[key];
				}
				let filter = {
					label: convertFieldNameToLabel(key),
					fieldname: key,
					filter_type: filterType,
					value,
				};
				filters.push(filter);
			}
			return filters;
		};

		const manager = ref({
			options,

			list,
			loading,
			totalCount,

			reload,
			update,

			getPage,
			nextPage,
			previousPage,
			hasNextPage,
			hasPreviousPage,
			currentPageNumber,
			totalPages,

			sudoFilters,
			addFilters,

			toggleOrderBy,

			onClick,

			select,
			unselect,
			selectAll,

			selectionMode,
			selectedItems,
			allItemsSelected,
			itemSelected,

			helperMethods: {
				convertFieldNameToLabel,
			},
		});
		provide("manager", manager);

		onMounted(() => {
			nextTick(() => {
				if (
					options.value.urlQueryFilters &&
					Object.keys(route.query).length > 0
				) {
					sudoFilters.value = convertUrlQueryToFilters();
					addFilters(sudoFilters.value, false);
				} else if (options.value.saveFiltersLocally) {
					sudoFilters.value = JSON.parse(
						localStorage.getItem(`list_filters_${route.path}`) || "[]"
					);
					addFilters(sudoFilters.value, options.value.urlQueryFilters);
				}
				reload();
			});
		});

		return {
			manager,
		};
	},
};
</script>
