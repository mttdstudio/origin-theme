<template>
  <main>
    <!-- Core product content -->
    <section class="pb-12 md:flex">
      <div class="relative md:w-1/2">
        <!-- Media slider for small screens -->
        <MediaSlider
          v-if="productImages"
          :media="productImages"
          class="h-0 md:hidden pb-full"
        />
        <!-- Fallback image -->
        <div
          v-else
          class="relative rounded md:hidden bg-primary-lighter pb-full"
        >
          <BaseIcon
            icon="uil:camera-slash"
            size="lg"
            class="absolute center-xy text-primary-med"
          />
        </div>
        <!-- Media stack for large screens -->
        <div class="hidden h-full md:block">
          <div v-if="$fetchState.pending" class="h-full bg-primary-lighter" />
          <template v-else>
            <div v-if="productImages">
              <VisualMedia
                v-for="image in productImages"
                :key="image.id"
                :source="image"
                :alt="image.alt"
                sizes="(min-width: 768px) 50vw, 100vw"
              />
            </div>

            <!-- Fallback image -->
            <div v-else class="relative rounded bg-primary-lighter pb-full">
              <BaseIcon
                icon="uil:camera-slash"
                size="lg"
                class="absolute center-xy text-primary-med"
              />
            </div>
          </template>
        </div>

        <!-- Back button -->
        <a
          href="#"
          class="
            fixed
            flex
            items-center
            justify-center
            rounded-full
            shadow-md
            left-6
            bottom-6
            w-9
            h-9
            bg-primary-lighter
          "
          @click.prevent="navigateBack"
        >
          <BaseIcon icon="uil:angle-left" class="-ml-px" />
        </a>
      </div>

      <!-- Product overview -->
      <div class="md:w-1/2 lg:px-6 xl:px-12">
        <div
          class="
            container
            top-0
            pt-10
            transition-all
            duration-300
            ease-in-out
            max-w-160
            md:sticky md:pt-12
          "
          :class="headerIsVisible ? 'top-20' : 'top-0'"
        >
          <!-- Skeleton loader -->
          <div v-if="$fetchState.pending">
            <div class="w-32 h-3 mb-4 loader-el" />
            <div class="w-2/3 loader-el h-9 mb-7" />
            <div class="w-40 h-3 mb-4 loader-el" />
            <div class="w-20 h-4 mb-12 loader-el" />
            <div
              v-for="index in 7"
              :key="`skeleton-1-${index}`"
              :style="`width: ${100 - Math.random() * 20}%`"
              class="h-2 mb-4 loader-el"
            />
            <div class="flex justify-between mt-12 mb-4">
              <div class="w-24 h-3 loader-el" />
              <div class="w-48 h-3 loader-el" />
            </div>
            <div class="h-12 mb-10 loader-el" />
            <div
              v-for="index in 3"
              :key="`skeleton-2-${index}`"
              class="flex items-center mb-2"
            >
              <div class="w-5 h-5 mr-2 rounded-full loader-el" />
              <div
                :style="`width: ${80 - Math.random() * 30}%`"
                class="h-2 loader-el"
              />
            </div>
          </div>

          <!-- Main content -->
          <div v-else>
            <!--TODO<div class="mb-2 label-xs-bold text-primary-dark">{{ breadcrumb }}</div>-->
            <h1 class="mb-4 leading-tight">
              {{ product.name }}
            </h1>
            <!--TODO awaiting customer reviews feature
            <ReviewStars :score="reviews.averageScore" size="sm" />
            <span class="text-sm">{{ reviews.total }} reviews</span>
            -->
            <div
              class="flex items-center mt-2 mb-5 text-lg font-semibold md:mb-8"
            >
              <span>{{ formatMoney(variation.price, currency) }}</span>
              <span v-if="billingInterval" class="lowercase"
                >&nbsp;{{ billingInterval }}</span
              >
              <span
                v-if="variation.origPrice"
                class="
                  inline-block
                  h-6
                  px-2
                  ml-3
                  text-xs
                  leading-loose
                  uppercase
                  rounded
                  bg-error-faded
                  -mt-2px
                  text-error-default
                "
              >
                {{ $t('products.slug.save') }}
                {{
                  formatMoney(variation.origPrice - variation.price, currency)
                }}
              </span>
            </div>
            <div v-html="product.description" />

            <!-- Product options -->
            <div v-for="input in optionInputs" :key="input.name" class="my-8">
              <component
                :is="input.component"
                v-if="visibleOptionIds.includes(input.option.id)"
                :option="input.option"
                :current-value="optionState[input.option.name]"
                :active-dropdown-u-i-d="activeDropdownUID"
                :validation="$v.optionState[input.option.name]"
                @value-changed="setOptionValue"
                @dropdown-active="setActiveDropdownUID($event)"
              />
            </div>
            <!-- Purchase options -->
            <ProductPurchaseOptions
              v-if="product.purchaseOptions"
              v-model="selectedPurchaseOption"
              :options="product.purchaseOptions"
              :option-state="optionState"
              :product="product"
              :quantity="quantity"
            />

            <!-- Cart button & stock info -->
            <div v-if="variation" class="relative my-8">
              <StockStatus
                v-if="product.stockTracking && !product.stockPurchasable"
                :status-value="variation.stockStatus"
              />

              <!-- Quantity -->
              <div class="flex">
                <ProductQuantity
                  v-if="enableQuantity"
                  v-model="quantity"
                  :initial-limit="maxQuantity"
                  :stock-tracking="variation.stockTracking"
                  :stock-purchasable="variation.stockPurchasable"
                  :stock-level="variation.stockLevel"
                />

                <!-- Add to cart -->
                <button
                  :class="{
                    loading: cartIsUpdating,
                    disabled: !stockAvailable,
                  }"
                  type="submit"
                  class="relative w-full btn btn--lg"
                  :disabled="!stockAvailable"
                  @click.prevent="addToCart"
                >
                  <div v-show="!cartIsUpdating">
                    <span>{{ $t('products.slug.addToCart') }}</span>
                    <span
                      class="
                        inline-block
                        w-5
                        mx-1
                        mb-1
                        border-b border-primary-lightest
                      "
                    />
                    <span>{{
                      formatMoney(variation.price * quantity, currency)
                    }}</span>
                    <span v-if="billingInterval">{{ billingInterval }}</span>
                    <span
                      v-if="variation.origPrice"
                      class="ml-1 line-through text-primary-med"
                    >
                      {{
                        formatMoney(variation.origPrice * quantity, currency)
                      }}
                    </span>
                    <span
                      v-if="
                        selectedPurchaseOption &&
                        selectedPurchaseOption.type === 'subscription'
                      "
                      class="lowercase"
                    >
                      / {{ intervalCount }}{{ subscriptionInterval }}
                    </span>
                  </div>
                  <div v-show="cartIsUpdating" class>
                    <div class="absolute inset-0 mt-3 spinner" />
                    <span class="absolute inset-0 mt-5">{{
                      $t('products.slug.updating')
                    }}</span>
                  </div>
                </button>
              </div>
            </div>
            <!-- END Purchase form -->

            <!-- Store features -->
            <div class="my-8">
              <ul class>
                <li
                  v-for="(benefit, index) in productBenefits"
                  :key="'storeProductBenefit' + index"
                  class="flex my-2 label-sm"
                >
                  <BaseIcon :icon="benefit.icon" size="sm" class="mr-2 -mb-1" />
                  <span>{{ benefit.text }}</span>
                </li>
              </ul>
            </div>

            <!-- Details & attributes -->
            <div v-for="attribute in product.attributes" :key="attribute.id">
              <template v-if="attribute.visible">
                <component
                  :is="getAttributeComponent(attribute.type)"
                  :attribute="attribute"
                />
              </template>
            </div>

            <!-- Share product -->
            <div v-if="enableSocialSharing" class="flex flex-no-wrap py-3">
              <strong class="w-1/4 pr-6 text-primary-darkest">{{
                $t('products.slug.share')
              }}</strong>
              <div class="flex justify-end w-3/4">
                <SocialShare
                  class="mr-2 cursor-pointer"
                  network="facebook"
                  :url="pageMeta.url"
                  :title="pageMeta.title"
                  :description="pageMeta.description"
                >
                  <BaseIcon icon="mdi:facebook" />
                </SocialShare>

                <SocialShare
                  class="mr-2 cursor-pointer"
                  network="twitter"
                  :url="pageMeta.url"
                  :title="pageMeta.title"
                  :description="pageMeta.description"
                >
                  <BaseIcon icon="mdi:twitter" />
                </SocialShare>

                <SocialShare
                  class="mr-2 cursor-pointer"
                  network="pinterest"
                  :url="pageMeta.url"
                  :description="pageMeta.description"
                  :media="pageMeta.image"
                >
                  <BaseIcon icon="mdi:pinterest" />
                </SocialShare>

                <SocialShare
                  class="cursor-pointer"
                  network="email"
                  :url="pageMeta.url"
                  :title="pageMeta.title"
                  :description="pageMeta.description"
                >
                  <BaseIcon icon="mdi:envelope" />
                </SocialShare>
              </div>
            </div>
          </div>
        </div>
      </div>
    </section>
  </main>
</template>

<script>
// Helpers
import { mapState } from 'vuex'
import get from 'lodash/get'
import { validationMixin } from 'vuelidate'
import { required } from 'vuelidate/lib/validators'
import pageMeta from '~/mixins/pageMeta'
import { listVisibleOptions } from '~/modules/swell'
import { getInitialSelection } from '~/utils/purchaseOptions'

export default {
  name: 'ProductDetailPage',
  mixins: [pageMeta, validationMixin],

  data() {
    return {
      product: {},
      quantity: 1,
      enableQuantity: true,
      maxQuantity: 99,
      relatedProducts: [], // TODO
      optionState: null,
      selectedPurchaseOption: undefined,
      productBenefits: [],
      enableSocialSharing: false,
      activeDropdownUID: null,
    }
  },

  async fetch() {
    const { $swell, $route } = this

    // Fetch product
    const product = await $swell.products.get($route.params.slug)

    // Show 404 if product isn't found
    if (!product) {
      return this.$nuxt.error({
        statusCode: 404,
        message: this.$t('errors.productNotFound'),
      })
    }

    // Compute initial values for options
    const optionState =
      this.optionState ||
      (product.options || []).reduce((options, { name, values, inputType }) => {
        // Set first available value for select current option
        if (!inputType || inputType === 'select') {
          options[name] = get(values, '0.name')
        }
        return options
      }, {})

    // TODO generate related products
    const relatedProducts = []
    let maxQuantity = get(product, 'content.maxQuantity')
    maxQuantity = !maxQuantity
      ? 99
      : typeof maxQuantity === 'string'
      ? Number(maxQuantity)
      : 99
    maxQuantity = !isNaN(maxQuantity) ? maxQuantity : 99

    this.selectedPurchaseOption = getInitialSelection(product.purchaseOptions)

    // Set component data
    this.product = product
    this.optionState = optionState
    this.relatedProducts = relatedProducts
    this.productBenefits = get(product, 'content.productBenefits', [])
    this.enableSocialSharing = get(product, 'content.enableSocialSharing')
    this.enableQuantity = get(product, 'content.enableQuantity')
    this.maxQuantity = maxQuantity
  },

  computed: {
    ...mapState(['cartIsUpdating', 'headerIsVisible', 'currency']),

    // Resulting combination of selected product options
    variation() {
      if (!this.product) return {}
      return this.$swell.products.variation(
        this.product,
        this.optionState,
        this.selectedPurchaseOption
      )
    },

    productImages() {
      if (!this.product?.images?.length) return null
      return this.product.images
    },

    stockAvailable() {
      const { stockStatus, stockTracking, stockPurchasable } = this.variation
      return (
        (stockStatus && stockStatus !== 'out_of_stock') ||
        !stockTracking ||
        stockPurchasable
      )
    },

    billingInterval() {
      return get(this, 'optionState.Plan')
    },

    intervalData() {
      if (
        !this.selectedPurchaseOption ||
        this.selectedPurchaseOption.type !== 'subscription'
      ) {
        return
      }

      // Placeholder until swell-js provides interval data inside variation()
      const currentPlan = this.product.purchaseOptions.subscription.plans.find(
        (plan) => plan.id === this.selectedPurchaseOption.plan
      )
      const { interval, intervalCount } = currentPlan.billingSchedule

      return { interval, intervalCount }
    },

    intervalCount() {
      if (!this.intervalData) return
      const { intervalCount } = this.intervalData
      return intervalCount > 1 ? intervalCount : ''
    },

    subscriptionInterval() {
      if (!this.intervalData) return
      return this.$t(
        `products.slug.purchaseOptions.interval.${this.intervalData.interval}.short`
      )
    },

    visibleOptionIds() {
      const options = get(this, 'product.options', [])
      const optionState = this.optionState

      return listVisibleOptions(options, optionState).map(({ id }) => id)
    },

    optionInputs() {
      const options = get(this, 'product.options', [])

      return options.reduce((optionInputs, option) => {
        let componentName

        switch (option.inputType) {
          case 'short_text':
            componentName = 'Text'
            break
          case 'long_text':
            componentName = 'Text'
            break
          case 'toggle':
            componentName = 'Checkbox'
            break
          default:
            componentName = 'Select'
        }

        // Don't include subscription plan if there's only one option value available
        if (option.subscription && option.values.length < 2) return optionInputs

        optionInputs.push({
          option,
          component: () =>
            import(`../../components/ProductOption${componentName}.vue`),
        })

        return optionInputs
      }, [])
    },
  },

  watch: {
    currency: '$fetch',
  },

  methods: {
    // Select attribute component based on type
    getAttributeComponent(type) {
      switch (type) {
        case 'long_text':
        case 'textarea':
          return 'AttributeLongText'
        case 'file':
          return 'AttributeFile'
        // TODO: add components for other supported attribute types
        default:
          return 'AttributeShortText'
      }
    },

    // Set which dropdown is active by UID, so that only one dropdown is active at any time.
    setActiveDropdownUID(uid) {
      this.activeDropdownUID = uid
    },

    // Add product to cart with selected options
    addToCart() {
      // Touch and validate all fields
      this.$v.$touch()
      if (this.$v.$invalid) return // return if invalid

      this.$store.dispatch('addCartItem', {
        productId: this.variation.id,
        quantity: this.quantity || 1,
        options: this.optionState,
        purchaseOption: this.selectedPurchaseOption,
      })
    },

    // Update an option value based on user input
    setOptionValue({ option, value }) {
      // Use $set to update the data object because options are dynamic
      // and optionState won't be reactive otherwise
      // TODO in Vue 3 this.optionState[option] = value should work
      this.$set(this.optionState, option, value)
    },

    // Go back to previous page
    navigateBack() {
      this.$router.back()
    },
  },

  validations() {
    const options = get(this, 'product.options', [])
    const fields = options.reduce((obj, option) => {
      if (option.required) {
        obj[option.name] = { required }
      }
      return obj
    }, {})

    return {
      optionState: fields,
    }
  },
}
</script>
