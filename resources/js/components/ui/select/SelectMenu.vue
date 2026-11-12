<script setup lang="ts" generic="T extends string | number | null">
import { Check, ChevronDown } from '@lucide/vue';
import {
    SelectContent,
    SelectGroup,
    SelectItem,
    SelectItemIndicator,
    SelectItemText,
    SelectLabel,
    SelectPortal,
    SelectRoot,
    SelectSeparator,
    SelectTrigger,
    SelectViewport,
} from 'reka-ui';
import { computed, nextTick, ref, useAttrs, watch } from 'vue';
import { cn } from '@/lib/utils';

export type SelectMenuOption<V> = {
    value: V;
    label: string;
    hint?: string;
    disabled?: boolean;
};

export type SelectMenuGroup<V> = {
    label: string;
    options: SelectMenuOption<V>[];
};

defineOptions({ inheritAttrs: false });

const props = defineProps<{
    /** Like a select's value: it can be looser than the options (5 or '5'). */
    modelValue: string | number | null | undefined;
    options: (SelectMenuOption<T> | SelectMenuGroup<T>)[];
    id?: string;
    name?: string;
    placeholder?: string;
    disabled?: boolean;
    required?: boolean;
    align?: 'start' | 'center' | 'end';
}>();

const emit = defineEmits<{
    'update:modelValue': [value: T];
    change: [value: T];
}>();
const attrs = useAttrs();
const trigger = ref<{ $el: HTMLElement } | null>(null);
const native = ref<HTMLSelectElement | null>(null);

/**
 * A dropdown in place of the browser's select. Options can be any string or
 * number, '' and null included (reka's select can't take an empty value), so
 * each gets a key by position and the model keeps the real value. Like a
 * select, '' and null count as the same choice, and so do 5 and '5'.
 */
const groups = computed(() => {
    let i = 0;

    return props.options.map((entry) =>
        'options' in entry
            ? {
                  label: entry.label,
                  options: entry.options.map((o) => ({ ...o, key: `o${i++}` })),
              }
            : {
                  label: null,
                  options: [{ ...entry, key: `o${i++}` }],
              },
    );
});

const flat = computed(() => groups.value.flatMap((g) => g.options));

const same = (a: unknown, b: unknown) =>
    a === b || String(a ?? '') === String(b ?? '');

/**
 * What was just picked, until the model catches up. A select shows the new
 * choice straight away, even when the page only saves it to the server and
 * waits for the new props.
 */
const picked = ref<{ value: T } | null>(null);
watch(
    () => props.modelValue,
    () => (picked.value = null),
);

const current = computed(() =>
    picked.value ? picked.value.value : props.modelValue,
);

const chosen = computed(() =>
    flat.value.find((o) => same(o.value, current.value)),
);

const key = computed({
    get: () => chosen.value?.key ?? '',
    set: (next: string) => {
        const option = flat.value.find((o) => o.key === next);
        if (option && !same(option.value, current.value)) {
            picked.value = { value: option.value };
            emit('update:modelValue', option.value);
            emit('change', option.value);
            // Clears a "Choose one." note from an earlier submit.
            nextTick(() =>
                native.value?.dispatchEvent(
                    new Event('change', { bubbles: true }),
                ),
            );
        }
    },
});

const triggerClass = computed(() =>
    cn(
        'group inline-flex min-w-0 cursor-pointer items-center gap-2 text-left outline-none disabled:cursor-not-allowed disabled:opacity-60',
        attrs.class
            ? null
            : 'h-9 w-full rounded-[8px] border border-(--ds-border) bg-(--ds-field) px-3 text-[13px]/5 text-(--ds-text) transition-colors hover:border-(--ds-border-hover) focus-visible:border-(--ds-ring) data-[state=open]:border-(--ds-border-hover)',
        attrs.class as string,
    ),
);
</script>

<template>
    <SelectRoot
        v-model="key"
        :disabled="disabled"
        :required="required"
    >
        <SelectTrigger
            :id="id"
            ref="trigger"
            v-bind="{ ...attrs, class: undefined }"
            :class="triggerClass"
            data-slot="select-menu-trigger"
        >
            <span
                class="min-w-0 flex-1 truncate"
                :class="{ 'text-(--ds-text-faint)': !chosen }"
                >{{ chosen?.label ?? placeholder ?? '' }}</span
            >
            <ChevronDown
                class="mi-drop size-3.5 shrink-0 text-(--ds-text-faint) group-data-[state=open]:rotate-180"
                :stroke-width="1.75"
            />
        </SelectTrigger>
        <SelectPortal>
            <SelectContent
                data-slot="select-content"
                position="popper"
                :align="align ?? 'start'"
                :side-offset="4"
                :collision-padding="8"
                class="z-50 max-h-[min(20rem,var(--reka-select-content-available-height))] min-w-(--reka-select-trigger-width) overflow-hidden font-ui data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:animate-in data-[state=open]:fade-in-0 data-[state=open]:zoom-in-[0.98] data-[side=bottom]:slide-in-from-top-1 data-[side=top]:slide-in-from-bottom-1"
            >
                <SelectViewport class="max-h-[inherit]">
                    <template
                        v-for="(group, g) in groups"
                        :key="group.options[0]?.key ?? g"
                    >
                        <SelectSeparator
                            v-if="group.label && g > 0"
                            data-slot="select-separator"
                        />
                        <SelectGroup>
                            <SelectLabel
                                v-if="group.label"
                                data-slot="select-label"
                                >{{ group.label }}</SelectLabel
                            >
                            <SelectItem
                                v-for="option in group.options"
                                :key="option.key"
                                :value="option.key"
                                :disabled="option.disabled"
                                :text-value="option.label"
                                data-slot="select-item"
                                class="relative flex cursor-pointer items-center px-2 pr-7 outline-none select-none data-disabled:cursor-default data-disabled:opacity-50"
                            >
                                <SelectItemText class="truncate">{{
                                    option.label
                                }}</SelectItemText>
                                <span
                                    v-if="option.hint"
                                    class="ml-auto shrink-0 pl-3 text-[12px] text-(--ds-text-faint)"
                                    >{{ option.hint }}</span
                                >
                                <SelectItemIndicator
                                    class="absolute right-2 flex items-center"
                                >
                                    <Check
                                        class="size-3.5 text-(--ds-text)"
                                        :stroke-width="2"
                                    />
                                </SelectItemIndicator>
                            </SelectItem>
                        </SelectGroup>
                    </template>
                </SelectViewport>
            </SelectContent>
        </SelectPortal>
    </SelectRoot>
    <!-- Sends the value with a plain form, and lets the browser check required. -->
    <select
        v-if="name || required"
        ref="native"
        :name="name"
        :required="required"
        :value="current ?? ''"
        tabindex="-1"
        aria-hidden="true"
        class="sr-only"
        @focus="trigger?.$el.focus()"
    >
        <option :value="current ?? ''">{{ chosen?.label ?? '' }}</option>
    </select>
</template>
