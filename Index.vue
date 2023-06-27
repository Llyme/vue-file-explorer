<script setup lang="ts">
declare const ipc: any;

import { ref, computed, toRaw } from 'vue';
import { Singleton, singletons as s } from '@/lib/renderer/singleton';
import strings from '@/strings.json';

import back_48 from '@/images/back.png';
import more_48 from '@/images/more.png';
import toggle_48 from '@/images/toggle_all.png';
import untoggle_48 from '@/images/untoggle_all.png';

import Gap from '@/components/Gap.vue';
import WindowVue from '@/components/Window/Window.vue';
import SubmitVue from '@/components/SuperButton/Submit.vue';
import CancelVue from '@/components/SuperButton/Cancel.vue';
import SuperButton from '@/components/SuperButton/SuperButton.vue';
import ListItem from '@/components/ListItem/ListItem.vue';
import ListItemRow from '@/components/ListItem/Row.vue';
import SuperImage from '@/components/SuperImage.vue';
import CheckboxInput from '../CheckboxInput.vue';

type Item = {
	name: string,
	base_path: BasePathType,
	is_file: boolean
	fullpath?: string,
	image?: string,
};

enum BasePathType {
	App = 'app',
	Local = 'local'
}

type SelectablePredicate = (item: Item) => boolean;
type FilterPredicate = (item: Item) => boolean;
type ShowResolve = (item: Item[]) => void;

const $window = ref();
const $addressPathSplit = ref<string[]>([]);
const $items = ref<Item[]>([]);
const $selectedItems = ref<Item[]>([]);
const $multiSelect = ref<boolean>(false);
const $basePath = ref<BasePathType|null>(null);
/**
 * If the file explorer should be locked to a base path or not.
 */
const $lockBasePath = ref<BasePathType|null>(null);
const $selectablePredicate = ref<SelectablePredicate>(null);
const $filterPredicate = ref<FilterPredicate>(null);
const $currentResolve = ref<ShowResolve>(null);
/**
 * If this is doing something.
 */
const $busy = ref<boolean>(false);


const $Items = computed({
	get() {
		return $items.value;
	},

	set(value) {
		$items.value = value;
		$selectedItems.value = [];
	}
});
const $Visible = computed({
	get() {
		return $window.value.$view.$dim.$visible;
	},

	set(value) {
		$window.value.$view.$dim.$visible = value;
	}
});
const $AddressPath = computed({
	get() {
		return $addressPathSplit.value.join('\\');
	},

	set(value) {
		if ($busy.value)
			throw new Error('[FileExplorer] Busy.');

		$addressPathSplit.value =
			value != null
			? value.match(/[^\\/]+/g) ?? []
			: [];

		Redraw();
	}
});


function Select(item: Item, checked: boolean) {
	if (checked) {
		if (!$selectedItems.value.includes(item))
			$selectedItems.value.push(item);

		return;
	}

	let index = $selectedItems.value.indexOf(item);

	if (index == -1)
		return;

	$selectedItems.value.splice(index, 1);
}

function IsSelectable(item: Item) {
	if ($selectablePredicate.value == null)
		return true;

	return $selectablePredicate.value(item);
}

async function Redraw() {
	if ($basePath.value == null) {
		$Items.value = [{
			name: 'App',
			base_path: BasePathType.App,
			is_file: false
		}, {
			name: 'Local',
			base_path: BasePathType.Local,
			is_file: false
		}];

		return;
	}

	let address_path = $AddressPath.value;
	let entries_result = null;


	$busy.value = true;

	switch ($basePath.value) {
		case BasePathType.App:
			entries_result = await ipc.file_explorer.GetApp(address_path);
			break;

		case BasePathType.Local:
			entries_result = await ipc.file_explorer.GetLocal(address_path);
			break;
	}

	$busy.value = false;


	if (entries_result == null)
		throw new Error('[FileExplorer] Illegal invocation.');

	if (entries_result.code != 200)
		return;

	let dirents =
		entries_result.res
		.map((entry: any) => ({
			...entry,
			
			base_path: $basePath.value
		}));

	if ($filterPredicate.value != null)
		dirents = dirents.filter($filterPredicate.value);

	$items.value = dirents;
}

async function Back() {
	if ($busy.value)
			throw new Error('[FileExplorer] Busy.');

	if (!$addressPathSplit.value.length) {
		if ($lockBasePath.value != null)
			throw new Error('[FileExplorer] No where to go back to.');

		$basePath.value = null;

		await Redraw();

		return;
	}

	$addressPathSplit.value.pop();

	await Redraw();
}

async function Navigate(index: number) {
	while ($addressPathSplit.value.length - 1 > index)
		$addressPathSplit.value.pop();

	await Redraw();
}

function Open(item: Item) {
		$basePath.value = item.base_path;
		$AddressPath.value = item.fullpath;
}

function Click(item: Item) {
	if (!item.is_file)
		return Open(item);

	if (IsSelectable(item))
		Select(item, !$selectedItems.value.includes(item));

	if (!$multiSelect.value)
		Submit();
}

function Submit() {
	$currentResolve.value(toRaw($selectedItems.value));

	$currentResolve.value = null;
	$Visible.value = false;
}

function Cancel() {
	$currentResolve.value([]);

	$currentResolve.value = null;
	$Visible.value = false;
}

function Options(event: CustomEvent, item: Item) {
	s.ContextMenu.Clear();

	if (IsSelectable(item))
		s.ContextMenu.Add({
			text: strings.select,
			color: 'green',

			callback() {
				Select(item, !$selectedItems.value.includes(item));

				if (!$multiSelect.value)
					Submit();

				return true;
			}
		});

	if (!item.is_file)
		s.ContextMenu.Add({
			text: strings.open,

			callback() {
				Open(item);

				return true;
			}
		});

	s.ContextMenu.Add({
		text: strings.cancel,
		color: 'gray'
	});

	s.ContextMenu.ShowFromEvent(event.detail.down_event);
}


defineExpose({
	$multiSelect,
	$addressPathSplit,
	$window,
	$selectedItems,
	$basePath,
	$lockBasePath,
	$selectablePredicate,

	$Busy: computed(() => $busy.value),
	$Items,
	$AddressPath,

	Reset() {
		$multiSelect.value = false;
		$Items.value = [];
		$basePath.value = null;
		$lockBasePath.value = null;
		$selectablePredicate.value = null;
		$AddressPath.value = '';
	},

	Show(): Promise<Item[]> {
		if ($currentResolve.value != null)
			$currentResolve.value([]);

		return new Promise(resolve => {
			$currentResolve.value = resolve;
			$Visible.value = true;
		});
	}
});

defineOptions(Singleton({
	name: 'FileExplorer',

	BasePathType
}));
</script>

<template>
  <WindowVue
    ref="$window"
    class="window relative"
    :max-width="720"
    :max-height="null"
    :visible="false"
    :disabled="$busy ? true : null"
  >
    <template #content>
      <div class="flex flex-row gap-[16px] h-[48px] flex-fixed m-[16px]">
        <SuperButton
          v-if="$addressPathSplit.length ||
            $lockBasePath == null && $basePath != null"
          color="blue"
          :image="back_48"
          :tooltip="strings.go_back"
          @click="Back()"
        />

        <div class="flex flex-auto flex-row gap-[8px]">
          <SuperButton
            v-for="(text, index) in $addressPathSplit"
            :key="index"
            :small="true"
            :disabled="index == $addressPathSplit.length - 1"
            class="flex-fixed self-center"
            color="gray"
            @click="Navigate(index)"
          >
            {{ text }}
          </SuperButton>
        </div>
      </div>

      <div class="flex flex-auto flex-col overflow-y-auto pointer-events-auto">
        <ListItem
          v-for="(item, index) in $items"
          :key="index"
          :interactive="true"
          @pointer-press="Click(toRaw(item))"
        >
          <ListItemRow
            class="gap-[16px]"
          >
            <CheckboxInput
              v-if="$multiSelect && IsSelectable(item)"
              class="self-center"
              :checked="$selectedItems.includes(item)"
              :tooltip="strings.select_unselect"
              @input="Select(item, $event.target.checked)"
            />

            <SuperImage
              v-if="item.image != null"
              class="w-[96px] h-[48px] flex-fixed self-center"
            />

            <label
              class="font-medium self-center
							one-liner flex-auto"
            >{{ item.name }}</label>

            <SuperButton
              class="self-center"
              color="blue"
              :image="more_48"
              :tooltip="strings.options"
              @pointer-press="Options($event, toRaw(item))"
            />
          </ListItemRow>
        </ListItem>
      </div>

      <div
        v-if="!$items.length"
        class="absolute top-0 left-0 w-full h-full flex flex-row"
      >
        <label
          class="self-center font-medium text-center w-full
					text-xl"
        >
          {{ strings.empty_directory }}
        </label>
      </div>

      <div
        class="mask absolute top-0 left-0 w-full h-full flex flex-row
				-white opacity-0 transition-opacity duration-200"
        :busy="$busy ? true : null"
      >
        <label
          class="self-center font-medium text-center w-full
					text-xl"
        >
          {{ strings.loading }}
        </label>
      </div>
    </template>

    <template #buttons>
      <SubmitVue
        tooltip="Select"
        :disabled="$busy || !$selectedItems.length"
        @click="Submit"
      />

      <Gap
        v-if="$multiSelect"
        :size="32"
      />

      <SuperButton
        v-if="$multiSelect && $selectedItems.length < $items.length"
        :fixed-size="48"
        :image="toggle_48"
        :circle="true"
        :tooltip="strings.select_all"
        :disabled="$busy"
        @click="$selectedItems = [...$items];"
      />

      <SuperButton
        v-if="$multiSelect && $selectedItems.length == $items.length"
        :fixed-size="48"
        :image="untoggle_48"
        :circle="true"
        :tooltip="strings.unselect_all"
        :disabled="$busy"
        @click="$selectedItems = [];"
      />

      <Gap
        :fill="true"
        :fixed="false"
      />

      <CancelVue
        :tooltip="strings.cancel"
        :disabled="$busy"
        @click="Cancel"
      />
    </template>
  </WindowVue>
</template>

<style scoped>
@import '@/styles/flex.css';
@import '@/styles/frame.css';
@import '@/styles/behavior.css';
@import '@/styles/text.css';

@tailwind base;
@tailwind components;
@tailwind utilities;

.dim[visible="false"] > .workspace {
	@apply shrink;
}

.window[disabled] * {
	pointer-events: none !important;
}

.mask[busy] {
	opacity: 80%;
}
</style>
