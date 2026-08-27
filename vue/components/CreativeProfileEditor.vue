<template>
    <div>
        <v-alert v-if="!sections.length" type="info" text dense>
            {{ t('profileFormEmpty', 'Для этого типа креатива профиль не редактируется.') }}
        </v-alert>

        <v-expansion-panels v-else v-model="openPanels" multiple accordion flat>
            <v-expansion-panel v-for="(section, index) in sections" :key="index">
                <v-expansion-panel-header>
                    <div class="d-flex align-center">
                        <span class="text-subtitle-1 font-weight-medium">{{ sectionTitle(section) }}</span>
                        <v-chip class="ml-3" x-small label :color="sectionColor(section)" outlined>
                            {{ filledCount(section) }} / {{ section.fields.length }}
                        </v-chip>
                        <v-icon v-if="sectionHasError(section)" small color="error" class="ml-2">
                            mdi-alert-circle
                        </v-icon>
                    </div>
                </v-expansion-panel-header>

                <v-expansion-panel-content>
                    <v-row dense>
                        <v-col v-for="field in section.fields" :key="field.name" cols="12" :md="colWidth(field)">
                            <!-- Закрытый словарь: своего не добавить -->
                            <v-select
                                v-if="field.control === 'Select'"
                                :value="valueOf(field)"
                                :items="field.options"
                                :label="fieldLabel(field)"
                                :error-messages="errorFor(field)"
                                :hint="hintFor(field)"
                                :disabled="disabled"
                                persistent-hint
                                clearable
                                outlined
                                dense
                                @change="setValue(field, $event)"
                            ></v-select>

                            <!-- Словарь длинный: значение набирают, а не выискивают в списке. -->
                            <v-autocomplete
                                v-else-if="field.control === 'MultiSelect'"
                                :ref="`suggested-${field.name}`"
                                :value="arrayValueOf(field)"
                                :items="field.options"
                                :label="fieldLabel(field)"
                                :error-messages="errorFor(field)"
                                :hint="hintFor(field)"
                                :disabled="disabled"
                                multiple
                                chips
                                small-chips
                                deletable-chips
                                persistent-hint
                                outlined
                                dense
                                @focus="showSuggestions(field)"
                                @change="setValue(field, $event)"
                            ></v-autocomplete>

                            <!-- Свободный ввод: подсказки не ограничивают, но нормализация обязательна -->
                            <v-combobox
                                v-else-if="field.control === 'Combo'"
                                :ref="`suggested-${field.name}`"
                                :value="arrayValueOf(field)"
                                :items="field.suggestions"
                                :label="fieldLabel(field)"
                                :error-messages="errorFor(field)"
                                :hint="hintFor(field)"
                                :disabled="disabled"
                                multiple
                                chips
                                small-chips
                                deletable-chips
                                auto-select-first
                                persistent-hint
                                outlined
                                dense
                                @focus="showSuggestions(field)"
                                @input="setFreeList(field, $event)"
                            ></v-combobox>

                            <v-combobox
                                v-else-if="field.control === 'ComboSingle'"
                                :ref="`suggested-${field.name}`"
                                :value="valueOf(field)"
                                :items="field.suggestions"
                                :label="fieldLabel(field)"
                                :error-messages="errorFor(field)"
                                :hint="hintFor(field)"
                                :disabled="disabled"
                                auto-select-first
                                persistent-hint
                                clearable
                                outlined
                                dense
                                @focus="showSuggestions(field)"
                                @input="setFreeValue(field, $event)"
                            ></v-combobox>

                            <v-text-field
                                v-else-if="field.control === 'Number'"
                                :value="valueOf(field)"
                                :label="fieldLabel(field)"
                                :error-messages="errorFor(field)"
                                :hint="hintFor(field)"
                                :disabled="disabled"
                                type="number"
                                persistent-hint
                                clearable
                                outlined
                                dense
                                @change="setNumber(field, $event)"
                            ></v-text-field>

                            <v-alert
                                v-if="droppedValues(field).length"
                                type="warning"
                                class="mt-1 mb-0"
                                dense
                                text
                            >
                                {{ t('profileValuesDropped', 'Не по правилам ввода и убрано из формы:') }}
                                <strong>{{ droppedValues(field).join(', ') }}</strong>.
                                {{ t('profileValuesDroppedHint', 'Сохранение профиля закрепит это.') }}
                            </v-alert>
                        </v-col>
                    </v-row>
                </v-expansion-panel-content>
            </v-expansion-panel>
        </v-expansion-panels>
    </div>
</template>

<script>
// Латиница, цифры, пробел и дефис — тот же контракт, что проверяет сервер (ProfileValueNormalizer).
const ALLOWED = /^[a-z0-9]+([ -][a-z0-9]+)*$/;
const INTEGER = /^-?\d+$/;

export default {
    name: 'CreativeProfileEditor',

    props: {
        // Дескриптор с сервера: секции → поля → тип контрола → словарь либо подсказки.
        sections: { type: Array, default: () => [] },
        // Текущие значения профиля по именам полей.
        value: { type: Object, default: () => ({}) },
        // Ошибки с сервера по именам полей.
        fieldErrors: { type: Object, default: () => ({}) },
        disabled: { type: Boolean, default: false },
        /**
         * Перевод по устойчивому ключу: (key, fallback) => string. Дескриптор приходит с сервера
         * на русском, поэтому двуязычный фронт подменяет строки у себя, а одноязычный ничего
         * не передаёт и получает серверные как есть.
         */
        translate: { type: Function, default: null },
    },

    data() {
        return {
            openPanels: [],
            localErrors: {},
            dropped: {},
        };
    },

    watch: {
        sections: {
            immediate: true,
            handler(sections) {
                // Разделов немного — открываем все: так видно профиль целиком, без кликов.
                this.openPanels = (sections || []).map((_, index) => index);
                // Значения приходят тем же присваиванием, что и дескриптор, — ждём их применения.
                this.$nextTick(this.adopt);
            },
        },
    },

    methods: {
        t(key, fallback) {
            return this.translate ? this.translate(key, fallback) : fallback;
        },

        sectionTitle(section) {
            return this.t(`profileSection.${section.code}`, section.title);
        },

        fieldLabel(field) {
            return this.t(`profileField.${field.name}`, field.label);
        },

        valueOf(field) {
            const value = this.value[field.name];

            return Array.isArray(value) ? value[0] : value;
        },

        arrayValueOf(field) {
            return this.asList(this.value[field.name]);
        },

        asList(value) {
            if (Array.isArray(value)) return value;

            return value === null || value === undefined || value === '' ? [] : [value];
        },

        fields() {
            return this.sections.reduce((all, section) => all.concat(section.fields), []);
        },

        droppedValues(field) {
            return this.dropped[field.name] || [];
        },

        /**
         * Приводит загруженный профиль к правилам формы: значения вне закрытого словаря и всё, что
         * не проходит проверку свободного ввода, отбрасываются сразу и показываются отдельно.
         * Иначе профиль, собранный ИИ до появления словарей, выглядит в форме одним, а сервер
         * на сохранении отбивает поля, которых человек не касался.
         */
        adopt() {
            if (!this.sections.length) return;

            const adopted = { ...this.value };
            const dropped = {};
            let changed = false;

            this.fields().forEach((field) => {
                const source = this.value[field.name];

                if (source === null || source === undefined || source === '') return;

                const { value, rejected } = this.accept(field, source);

                if (rejected.length) dropped[field.name] = rejected;

                if (this.differs(source, value)) {
                    adopted[field.name] = value;
                    changed = true;
                }
            });

            this.dropped = dropped;

            if (changed) this.$emit('input', adopted);
        },

        /** Разбирает сохранённое значение поля: что форма принимает и что вынуждена отбросить. */
        accept(field, source) {
            if (field.control === 'Number') {
                const parsed = Number.parseInt(source, 10);

                return Number.isNaN(parsed) ? { value: null, rejected: [source] } : { value: parsed, rejected: [] };
            }

            const multiple = field.control === 'MultiSelect' || field.control === 'Combo';
            const closed = !!(field.options && field.options.length);
            const accepted = [];
            const rejected = [];

            this.asList(source).forEach((item) => {
                const parts = closed ? [item] : this.split(item);

                parts.forEach((part) => {
                    const value = closed ? this.matchOption(field, part) : this.normalize(part);

                    if (value === null) rejected.push(part);
                    else if (!accepted.includes(value)) accepted.push(value);
                });
            });

            if (multiple) return { value: accepted, rejected };

            return { value: accepted.length ? accepted[0] : null, rejected: rejected.concat(accepted.slice(1)) };
        },

        /** Закрытый словарь хранится в каноничном написании — сравниваем без учёта регистра. */
        matchOption(field, value) {
            const needle = String(value).trim().toLowerCase();

            return field.options.find((o) => String(o).toLowerCase() === needle) || null;
        },

        normalize(value) {
            const normalized = String(value).trim().replace(/\s+/g, ' ').toLowerCase();

            return normalized && ALLOWED.test(normalized) ? normalized : null;
        },

        /** Составное значение в одном поле ввода — это несколько значений, а не одно новое. */
        split(value) {
            return String(value)
                .split(/[,/]/)
                .filter((part) => part.trim());
        },

        /**
         * Форма массива против скаляра — тоже разница: сервер отбивает массив в одиночном поле
         * («Expected a single text value»), поэтому ["female"] обязано схлопнуться в "female".
         */
        differs(source, value) {
            if (Array.isArray(source) !== Array.isArray(value)) return true;

            if (Array.isArray(source)) {
                return source.length !== value.length || source.some((item, index) => item !== value[index]);
            }

            return source !== value;
        },

        /**
         * Открывает список подсказок при попадании в поле. Сами по себе они появляются только на
         * клик по стрелке или после первого набранного символа, а поле с чипами клик по свободному
         * месту не ловит — так подсказки и остаются незамеченными, а значения набирают заново.
         */
        showSuggestions(field) {
            const [control] = this.$refs[`suggested-${field.name}`] || [];

            if (control) control.activateMenu();
        },

        errorFor(field) {
            return this.localErrors[field.name] || this.fieldErrors[field.name] || [];
        },

        hintFor(field) {
            return field.hint ? this.t(`profileHint.${field.name}`, field.hint) : '';
        },

        colWidth(field) {
            return field.control === 'Number' ? 4 : 12;
        },

        filledCount(section) {
            return section.fields.filter((f) => {
                const value = this.value[f.name];

                return Array.isArray(value) ? value.length > 0 : value !== null && value !== undefined && value !== '';
            }).length;
        },

        sectionColor(section) {
            const filled = this.filledCount(section);

            if (filled === 0) return 'grey';

            return filled === section.fields.length ? 'success' : 'primary';
        },

        sectionHasError(section) {
            return section.fields.some((f) => this.errorFor(f).length);
        },

        setValue(field, value) {
            this.$set(this.localErrors, field.name, []);
            this.emit(field, value === undefined ? null : value);
        },

        setNumber(field, raw) {
            if (raw === null || raw === undefined || raw === '') {
                this.setValue(field, null);
                return;
            }

            // parseInt молча съедает «12.7» и «1e3»: у поля с градусами это тихо неверное значение.
            if (!INTEGER.test(String(raw).trim())) {
                this.$set(this.localErrors, field.name, [this.t('profileIntegerExpected', 'Ожидается целое число')]);
                return;
            }

            this.setValue(field, Number.parseInt(raw, 10));
        },

        setFreeValue(field, raw) {
            const { values, errors } = this.normalizeAll([raw], false);

            this.$set(this.localErrors, field.name, errors);
            this.emit(field, values.length ? values[0] : null);
        },

        setFreeList(field, raw) {
            const { values, errors } = this.normalizeAll(raw || [], true);

            this.$set(this.localErrors, field.name, errors);
            this.emit(field, values);
        },

        /**
         * Приводит ввод к тому же виду, что и сервер: обрезка, схлопывание пробелов, нижний регистр.
         * Непрошедшее значение не применяется, но применение остальных не отменяет — иначе в поле
         * остаётся чип, которого нет в модели, и сохранение молча теряет ввод. Поле, хранящее одно
         * значение, на части не режется: лишнее оттуда всё равно некуда деть, кроме как выбросить.
         */
        normalizeAll(list, multiple) {
            const values = [];
            const errors = [];

            for (const item of list) {
                if (item === null || item === undefined) continue;

                for (const part of multiple ? this.split(item) : [item]) {
                    const normalized = this.normalize(part);

                    if (normalized === null) {
                        const rule = this.t(
                            'profileValueRule',
                            'только латиница строчными буквами, цифры, пробел и дефис');

                        errors.push(`«${String(part).trim()}» — ${rule}`);
                        continue;
                    }

                    if (!values.includes(normalized)) values.push(normalized);
                }
            }

            return { values, errors };
        },

        emit(field, value) {
            this.$emit('input', { ...this.value, [field.name]: value });
        },
    },
};
</script>
