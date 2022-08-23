<template>
  <div id="home">
    <SfHero class="hero" v-if="!heroCarouselLoading">
      <SfHeroItem
        v-for="(hero, i) in heroCarouselContent"
        :key="i"
        :title="hero.title"
        :subtitle="hero.subtitle"
        :button-text="hero.buttonText"
        :background="hero.background"
        :image="formatImageOutput(hero.image)"
        :class="hero.className"
      />
    </SfHero>
    <LazyHydrate when-visible>
      <SfBannerGrid :banner-grid="1" class="banner-grid">
        <template v-for="item in banners" #[item.slot]>
          <SfBanner
            :key="item.slot"
            :title="item.title"
            :subtitle="item.subtitle"
            :description="item.description"
            :button-text="item.buttonText"
            :image="item.image"
            :class="item.class"
          />
        </template>
      </SfBannerGrid>
    </LazyHydrate>
    <LazyHydrate when-visible>
      <ProductsCarousel
        :products="carouselProducts"
        :loading="productCarouselLoadingAEM"
        title="Featured Products"
      />
    </LazyHydrate>

    <LazyHydrate when-visible>
      <SfCallToAction
        title="Subscribe to Newsletters"
        button-text="Subscribe"
        description="Be aware of upcoming sales and events. Receive gifts and special offers!"
        image="https://cdn.shopify.com/s/files/1/0407/1902/4288/files/newsletter_1240x202.jpg?v=1616496568"
        class="call-to-action"
      />
    </LazyHydrate>
    <LazyHydrate when-visible>
      <InstagramFeed />
    </LazyHydrate>

    <LazyHydrate when-visible>
      <MobileStoreBanner />
    </LazyHydrate>
  </div>
</template>
<script lang="ts" type="module">
import {
  SfHero,
  SfBanner,
  SfCallToAction,
  SfBannerGrid,
} from '@storefront-ui/vue';
import { useProduct } from '~/composables';
import {
  computed,
  defineComponent,
  useAsync,
  onBeforeMount,
  ref,
} from '@nuxtjs/composition-api';
import LazyHydrate from 'vue-lazy-hydration';
import MobileStoreBanner from '~/components/MobileStoreBanner.vue';
import InstagramFeed from '~/components/InstagramFeed.vue';
import ProductsCarousel from '~/modules/catalog/product/components/ProductsCarousel.vue';

// AEM:
import {
  useContent,
  getContentFragmentList,
  getContentFragment,
} from '@bounteous/vue-storefront-aem';

import { Logger } from '~/helpers/logger';

export default defineComponent({
  name: 'PageAem',
  components: {
    InstagramFeed,
    LazyHydrate,
    MobileStoreBanner,
    ProductsCarousel,
    SfBanner,
    SfBannerGrid,
    SfCallToAction,
    SfHero,
  },
  // eslint-disable-next-line @typescript-eslint/explicit-module-boundary-types
  setup() {
    // Hero Carousel AEM
    const {
      search: heroCarouselSearch,
      content: heroCarouselContent,
      loading: heroCarouselLoading,
      error: heroCarouselError,
    } = useContent('hero');

    // Get SKUs for the Product Carousel from AEM
    const {
      search: productCarouselSearchAEM,
      content: productCarouselContentAEM,
      loading: productCarouselLoadingAEM,
      error: productCarouselErrorAEM,
    } = useContent('productCarouselAEM');

    // And then query Magento with the provided SKUs
    const productCarouselResult = ref([]);
    const {
      getProductList: productCarouselSearchMagento,
      loading: productCarouselSearchLoading,
    } = useProduct('productCarouselMagento');

    // Workaround for nested image paths
    const formatImageOutput = (image) => {
      return {
        mobile: image?.mobile._publishUrl,
        desktop: image?.desktop._publishUrl,
      };
    };

    onBeforeMount(async () => {
      // Hero Carousel AEM
      const heroCarouselQuery = `
      {
        vsfHeroList {
          items {
            title
            subtitle
            buttonText
            background
            link
            image {
              mobile {
                ... on ImageRef {
                  _publishUrl
                }
              }
              desktop {
                ... on ImageRef {
                  _publishUrl
                }
              }
            }
          }
        }
      }
      `;
      await heroCarouselSearch({ query: heroCarouselQuery });

      Logger.debug('[AEM]:heroCarouselContent', heroCarouselContent.value);

      // Product Carousel AEM
      const productCarouselQuery = `
      {
        vsfCarouselProductsByPath(_path: "/content/dam/vsf-aem-demo/homepage-product-carousel") {
          item {
            products
          }
        }
      }
      `;
      await productCarouselSearchAEM({ query: productCarouselQuery });

      const productCarouselListMagento = await productCarouselSearchMagento({
        pageSize: 10,
        currentPage: 1,
        filter: {
          sku: {
            in: getContentFragment(productCarouselContentAEM.value)?.products,
          },
        },
      });

      Logger.debug('[AEM]:productCarouselListMagento', productCarouselListMagento?.items);

      productCarouselResult.value = productCarouselListMagento?.items;
    });

    return {
      heroCarouselContent: computed(() =>
        getContentFragmentList(heroCarouselContent.value)
      ),
      carouselProducts: computed(() => productCarouselResult.value),
      productCarouselLoadingAEM,
      heroCarouselLoading,
      formatImageOutput,
    };
  },
  // eslint-disable-next-line @typescript-eslint/explicit-module-boundary-types
  data() {
    return {
      banners: [
        {
          slot: 'banner-A',
          subtitle: 'Dresses',
          title: 'Cocktail & Party',
          description:
            "Find stunning women's cocktail dresses and party dresses. Stand out in lace and metallic cocktail dresses from all your favorite brands.",
          buttonText: 'Shop now',
          image: {
            mobile:
              'https://cdn.shopify.com/s/files/1/0407/1902/4288/files/bannerB_328x343.jpg',
            desktop:
              'https://cdn.shopify.com/s/files/1/0407/1902/4288/files/bannerF_332x840.jpg',
          },
          class: 'sf-banner--slim desktop-only',
          link: '/c/women/women-clothing-skirts',
        },
        {
          slot: 'banner-B',
          subtitle: 'Dresses',
          title: 'Linen Dresses',
          description:
            "Find stunning women's cocktail dresses and party dresses. Stand out in lace and metallic cocktail dresses from all your favorite brands.",
          buttonText: 'Shop now',
          image: {
            mobile:
              'https://cdn.shopify.com/s/files/1/0407/1902/4288/files/bannerE_328x343.jpg',
            desktop:
              'https://cdn.shopify.com/s/files/1/0407/1902/4288/files/bannerE_496x840.jpg',
          },
          class: 'sf-banner--slim banner-central desktop-only',
          link: '/c/women/women-clothing-dresses',
        },
        {
          slot: 'banner-C',
          subtitle: 'T-Shirts',
          title: 'The Office Life',
          image: {
            mobile:
              'https://cdn.shopify.com/s/files/1/0407/1902/4288/files/bannerC_328x343.jpg',
            desktop:
              'https://cdn.shopify.com/s/files/1/0407/1902/4288/files/bannerC_332x400.jpg',
          },
          class: 'sf-banner--slim banner__tshirt',
          link: '/c/women/women-clothing-shirts',
        },
        {
          slot: 'banner-D',
          subtitle: 'Summer Sandals',
          title: 'Eco Sandals',
          image: {
            mobile:
              'https://cdn.shopify.com/s/files/1/0407/1902/4288/files/bannerG_328x343.jpg',
            desktop:
              'https://cdn.shopify.com/s/files/1/0407/1902/4288/files/bannerG_332x400.jpg',
          },
          class: 'sf-banner--slim',
          link: '/c/women/women-shoes-sandals',
        },
      ],
    };
  },
});
</script>

<style lang="scss" scoped>
.article-meta h4 a {
  color: #111111;
  font-weight: 600;
  font-size: 20px;
}

.article-meta {
  margin-top: 10px;
}

.article-item__meta-item:not(:last-child)::after {
  display: inline-block;
  content: '';
  width: 5px;
  height: 5px;
  margin: -1px 10px 0 10px;
  border-radius: 100%;
  background: rgba(0, 0, 0, 0.4);
  vertical-align: middle;
}

#home {
  box-sizing: border-box;
  padding: 0 var(--spacer-sm);
  @include for-desktop {
    max-width: 1240px;
    padding: 0;
    margin: 0 auto;
  }
}

.hero {
  margin: var(--spacer-xl) auto var(--spacer-lg);
  --hero-item-background-position: center;

  ::v-deep .sf-link:hover {
    color: var(--c-white);
  }

  @include for-desktop {
    margin: var(--spacer-xl) auto var(--spacer-2xl);
  }

  .sf-hero-item {
    &:nth-child(even) {
      --hero-item-background-position: left;
      @include for-mobile {
        --hero-item-background-position: 30%;
        --hero-item-wrapper-text-align: right;
        --hero-item-subtitle-width: 100%;
        --hero-item-title-width: 100%;
        --hero-item-wrapper-padding: var(--spacer-sm) var(--spacer-sm)
          var(--spacer-sm) var(--spacer-2xl);
      }
    }
  }
}

::v-deep .sf-hero__controls {
  --hero-controls-display: none;
}

.banner-grid {
  --banner-container-width: 50%;
  margin: var(--spacer-xl) 0;

  ::v-deep .sf-link:hover {
    color: var(--c-white);
  }

  @include for-desktop {
    margin: var(--spacer-2xl) 0;
    ::v-deep .sf-link {
      --button-width: auto;
    }
  }
}

.banner {
  &__tshirt {
    background-position: left;
  }

  &-central {
    @include for-desktop {
      --banner-container-flex: 0 0 70%;
    }
  }
}

.similar-products {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-bottom: var(--spacer-2xs);
  --heading-padding: 0;
  border-bottom: 1px var(--c-light) solid;
  @include for-desktop {
    border-bottom: 0;
    justify-content: center;
    padding-bottom: 0;
  }
}

.call-to-action {
  background-position: right;
  margin: var(--spacer-xs) 0;
  @include for-desktop {
    margin: var(--spacer-xl) 0 var(--spacer-2xl) 0;
  }
}

.carousel {
  margin: 0 calc(var(--spacer-sm) * -1) 0 0;
  @include for-desktop {
    margin: 0;
  }

  &__item {
    margin: 1.375rem 0 2.5rem 0;
    @include for-desktop {
      margin: var(--spacer-xl) 0 var(--spacer-xl) 0;
    }

    &__product {
      --product-card-add-button-transform: translate3d(0, 30%, 0);
    }
  }
}
</style>
