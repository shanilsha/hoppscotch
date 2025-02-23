<template>
  <div>
    <div
      class="sticky z-10 flex items-center justify-between pl-4 border-b bg-primary border-dividerLight top-lowerSecondaryStickyFold"
    >
      <label class="font-semibold text-secondaryLight">{{
        t("response.body")
      }}</label>
      <div class="flex">
        <ButtonSecondary
          v-if="response.body"
          v-tippy="{ theme: 'tooltip' }"
          :title="t('state.linewrap')"
          :class="{ '!text-accent': linewrapEnabled }"
          svg="wrap-text"
          @click.native.prevent="linewrapEnabled = !linewrapEnabled"
        />
        <ButtonSecondary
          v-if="response.body"
          ref="downloadResponse"
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.download_file')"
          :svg="downloadIcon"
          @click.native="downloadResponse"
        />
        <ButtonSecondary
          v-if="response.body"
          ref="copyResponse"
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.copy')"
          :svg="copyIcon"
          @click.native="copyResponse"
        />
      </div>
    </div>
    <div ref="jsonResponse"></div>
    <div
      v-if="outlinePath"
      class="sticky bottom-0 z-10 flex flex-1 px-2 overflow-auto border-t bg-primaryLight border-dividerLight flex-nowrap hide-scrollbar"
    >
      <div
        v-for="(item, index) in outlinePath"
        :key="`item-${index}`"
        class="flex items-center"
      >
        <tippy
          ref="outlineOptions"
          interactive
          trigger="click"
          theme="popover"
          arrow
        >
          <template #trigger>
            <div v-if="item.kind === 'RootObject'" class="outline">{}</div>
            <div v-if="item.kind === 'RootArray'" class="outline">[]</div>
            <div v-if="item.kind === 'ArrayMember'" class="outline">
              {{ item.index }}
            </div>
            <div v-if="item.kind === 'ObjectMember'" class="outline">
              {{ item.name }}
            </div>
          </template>
          <div
            v-if="item.kind === 'ArrayMember' || item.kind === 'ObjectMember'"
          >
            <div v-if="item.kind === 'ArrayMember'" class="flex flex-col">
              <SmartItem
                v-for="(arrayMember, astIndex) in item.astParent.values"
                :key="`ast-${astIndex}`"
                :label="`${astIndex}`"
                @click.native="
                  () => {
                    jumpCursor(arrayMember)
                    outlineOptions[index].tippy().hide()
                  }
                "
              />
            </div>
            <div v-if="item.kind === 'ObjectMember'" class="flex flex-col">
              <SmartItem
                v-for="(objectMember, astIndex) in item.astParent.members"
                :key="`ast-${astIndex}`"
                :label="objectMember.key.value"
                @click.native="
                  () => {
                    jumpCursor(objectMember)
                    outlineOptions[index].tippy().hide()
                  }
                "
              />
            </div>
          </div>
          <div v-if="item.kind === 'RootObject'" class="flex flex-col">
            <SmartItem
              label="{}"
              @click.native="
                () => {
                  jumpCursor(item.astValue)
                  outlineOptions[index].tippy().hide()
                }
              "
            />
          </div>
          <div v-if="item.kind === 'RootArray'" class="flex flex-col">
            <SmartItem
              label="[]"
              @click.native="
                () => {
                  jumpCursor(item.astValue)
                  outlineOptions[index].tippy().hide()
                }
              "
            />
          </div>
        </tippy>
        <i
          v-if="index + 1 !== outlinePath.length"
          class="opacity-50 text-secondaryLight material-icons"
          >chevron_right</i
        >
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, reactive } from "@nuxtjs/composition-api"
import { useCodemirror } from "~/helpers/editor/codemirror"
import { HoppRESTResponse } from "~/helpers/types/HoppRESTResponse"
import jsonParse, { JSONObjectMember, JSONValue } from "~/helpers/jsonParse"
import { getJSONOutlineAtPos } from "~/helpers/newOutline"
import {
  convertIndexToLineCh,
  convertLineChToIndex,
} from "~/helpers/editor/utils"
import { useI18n } from "~/helpers/utils/composables"
import useCopyResponse from "~/helpers/lenses/composables/useCopyResponse"
import useResponseBody from "~/helpers/lenses/composables/useResponseBody"
import useDownloadResponse from "~/helpers/lenses/composables/useDownloadResponse"

const t = useI18n()

const props = defineProps<{
  response: HoppRESTResponse
}>()

const { responseBodyText } = useResponseBody(props.response)

const { copyIcon, copyResponse } = useCopyResponse(responseBodyText)

const { downloadIcon, downloadResponse } = useDownloadResponse(
  "application/json",
  responseBodyText
)

const jsonBodyText = computed(() => {
  try {
    return JSON.stringify(JSON.parse(responseBodyText.value), null, 2)
  } catch (e) {
    // Most probs invalid JSON was returned, so drop prettification (should we warn ?)
    return responseBodyText.value
  }
})

const ast = computed(() => {
  try {
    return jsonParse(jsonBodyText.value)
  } catch (_: any) {
    return null
  }
})

const outlineOptions = ref<any | null>(null)
const jsonResponse = ref<any | null>(null)
const linewrapEnabled = ref(true)

const { cursor } = useCodemirror(
  jsonResponse,
  jsonBodyText,
  reactive({
    extendedEditorConfig: {
      mode: "application/ld+json",
      readOnly: true,
      lineWrapping: linewrapEnabled,
    },
    linter: null,
    completer: null,
    environmentHighlights: true,
  })
)

const jumpCursor = (ast: JSONValue | JSONObjectMember) => {
  const pos = convertIndexToLineCh(jsonBodyText.value, ast.start)
  pos.line--
  cursor.value = pos
}

const outlinePath = computed(() => {
  if (ast.value) {
    return getJSONOutlineAtPos(
      ast.value,
      convertLineChToIndex(jsonBodyText.value, cursor.value)
    )
  } else return null
})
</script>

<style lang="scss" scoped>
.outline {
  @apply cursor-pointer;
  @apply flex-grow-0 flex-shrink-0;
  @apply text-secondaryLight;
  @apply inline-flex;
  @apply items-center;
  @apply px-2;
  @apply py-1;
  @apply transition;
  @apply hover:text-secondary;
}
</style>
