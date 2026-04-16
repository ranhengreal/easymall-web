<template>
  <div class="product-list-page">
    <div class="container">
      <!-- 面包屑 -->
      <el-breadcrumb separator="/" class="breadcrumb">
        <el-breadcrumb-item :to="{ path: '/' }">首页</el-breadcrumb-item>
        <el-breadcrumb-item>商品列表</el-breadcrumb-item>
        <el-breadcrumb-item v-if="currentCategoryName">{{ currentCategoryName }}</el-breadcrumb-item>
      </el-breadcrumb>

      <div class="list-wrapper">
        <!-- 侧边栏 - 分类筛选 -->
        <div class="sidebar">
          <div class="filter-box">
            <h3>商品分类</h3>
            <div class="category-tree">
              <CategoryTreeNode
                  v-for="cat in categories"
                  :key="cat.categoryId"
                  :category="cat"
                  :current-category-id="currentCategoryId"
                  @select="selectCategory"
              />
            </div>
          </div>
        </div>

        <!-- 主内容区 -->
        <div class="main-content">
          <!-- 搜索栏 -->
          <div class="search-bar">
            <el-input
                v-model="keyword"
                placeholder="搜索商品名称"
                prefix-icon="Search"
                size="large"
                clearable
                @keyup.enter="handleSearch"
                @clear="handleSearch"
            />
            <el-button type="primary" size="large" @click="handleSearch">搜索</el-button>
          </div>

          <!-- 排序栏 + 结果统计 -->
          <div class="sort-bar">
            <div class="sort-left">
              <div
                  v-for="sortOption in sortOptions"
                  :key="sortOption.value"
                  class="sort-item"
                  :class="{ active: orderBy === sortOption.value }"
                  @click="changeOrder(sortOption.value)"
              >
                {{ sortOption.label }}
                <el-icon v-if="sortOption.icon === 'top'"><Top /></el-icon>
                <el-icon v-if="sortOption.icon === 'bottom'"><Bottom /></el-icon>
              </div>
            </div>
            <div class="sort-right">
              <span class="result-count">共 {{ total }} 件商品</span>
            </div>
          </div>

          <!-- 商品网格 -->
          <div v-loading="loading" class="product-grid">
            <div
                v-for="product in displayProducts"
                :key="product.productId"
                class="product-card"
                @click="goDetail(product.productId)"
            >
              <div class="product-image">
                <img
                    :src="getImageUrl(product.mainImage)"
                    :alt="product.productName"
                    loading="lazy"
                    @error="handleImageError"
                >
                <!-- 商品标签 -->
                <div class="product-tags">
                  <span v-if="product.sales > 100" class="tag hot">热卖</span>
                  <span v-if="product.isNew" class="tag new">新品</span>
                </div>
              </div>
              <div class="product-info">
                <h3 class="product-title">{{ product.productName }}</h3>
                <div class="product-price">
                  <span class="price">¥{{ product.price?.toFixed(2) }}</span>
                  <span v-if="product.originalPrice" class="original-price">
                    ¥{{ product.originalPrice?.toFixed(2) }}
                  </span>
                </div>
                <div class="product-meta">
                  <span class="sales">月销 {{ formatSales(product.sales) }}</span>
                  <span class="stock" :class="{ 'low-stock': product.stock < 10 }">
                    {{ product.stock > 0 ? `库存 ${product.stock}` : '缺货' }}
                  </span>
                </div>
              </div>
            </div>
          </div>

          <!-- 空状态 -->
          <el-empty v-if="!loading && displayProducts.length === 0" description="暂无商品">
            <el-button type="primary" @click="resetFilters">重置筛选</el-button>
          </el-empty>

          <!-- 分页 -->
          <div v-if="total > pageSize" class="pagination">
            <el-pagination
                v-model:current-page="pageNum"
                v-model:page-size="pageSize"
                :total="total"
                :page-sizes="[12, 24, 48, 96]"
                layout="total, sizes, prev, pager, next, jumper"
                @current-change="handlePageChange"
                @size-change="handleSizeChange"
            />
          </div>
        </div>
      </div>
    </div>

    <!-- 返回顶部按钮 -->
    <el-backtop :right="40" :bottom="40" />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch, computed, nextTick } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import { Top, Bottom } from '@element-plus/icons-vue'
import { getProductList } from '@/api/product'
import { getCategoryTree } from '@/api/category'
import { getImageUrl, handleImageError } from '@/utils/image'
import CategoryTreeNode from '@/components/CategoryTreeNode.vue'
import type { Product, Category } from '@/api/product'

const route = useRoute()
const router = useRouter()

// 数据
const loading = ref(false)
const productList = ref<Product[]>([])
const categories = ref<Category[]>([])
const total = ref(0)

// 参数
const pageNum = ref(1)
const pageSize = ref(12)
const currentCategoryId = ref('')
const currentCategoryName = ref('')
const keyword = ref('')
const orderBy = ref('default')

// 排序选项配置
const sortOptions = [
  { label: '默认', value: 'default', icon: null },
  { label: '价格升序', value: 'price_asc', icon: 'top' },
  { label: '价格降序', value: 'price_desc', icon: 'bottom' },
  { label: '销量', value: 'sales', icon: null }
]

// 格式化销量
const formatSales = (sales: number) => {
  if (!sales) return '0'
  if (sales >= 10000) return `${(sales / 10000).toFixed(1)}万`
  return sales.toString()
}

// 重置筛选
const resetFilters = () => {
  currentCategoryId.value = ''
  currentCategoryName.value = ''
  keyword.value = ''
  orderBy.value = 'default'
  pageNum.value = 1
}

// 递归获取所有子分类ID（优化版）
const getAllChildCategoryIds = (list: Category[], parentId: string): string[] => {
  const findCategory = (cats: Category[], id: string): Category | null => {
    for (const cat of cats) {
      if (cat.categoryId === id) return cat
      if (cat.children?.length) {
        const found = findCategory(cat.children, id)
        if (found) return found
      }
    }
    return null
  }

  const collectIds = (category: Category): string[] => {
    let ids = [category.categoryId]
    if (category.children?.length) {
      for (const child of category.children) {
        ids = ids.concat(collectIds(child))
      }
    }
    return ids
  }

  const target = findCategory(list, parentId)
  if (target) {
    // 同时保存分类名称
    currentCategoryName.value = target.categoryName
    return collectIds(target)
  }
  return [parentId]
}

// 获取当前选中的分类及其所有子分类ID（优化版）
const selectedCategoryIds = computed(() => {
  if (!currentCategoryId.value) return []
  return getAllChildCategoryIds(categories.value, currentCategoryId.value)
})

// 前端筛选（合并为一个 computed，减少中间数组创建）
const filteredProducts = computed(() => {
  let result = [...productList.value]

  // 分类筛选
  if (selectedCategoryIds.value.length > 0) {
    result = result.filter(p => selectedCategoryIds.value.includes(p.categoryId))
  }

  // 关键词搜索
  if (keyword.value.trim()) {
    const kw = keyword.value.toLowerCase().trim()
    result = result.filter(p => p.productName.toLowerCase().includes(kw))
  }

  // 排序
  switch (orderBy.value) {
    case 'price_asc':
      result.sort((a, b) => (a.price || 0) - (b.price || 0))
      break
    case 'price_desc':
      result.sort((a, b) => (b.price || 0) - (a.price || 0))
      break
    case 'sales':
      result.sort((a, b) => (b.sales || 0) - (a.sales || 0))
      break
    default:
      result.sort((a, b) => {
        const timeA = a.createTime ? new Date(a.createTime).getTime() : 0
        const timeB = b.createTime ? new Date(b.createTime).getTime() : 0
        return timeB - timeA
      })
      break
  }

  return result
})

// 分页后的数据
const displayProducts = computed(() => {
  const start = (pageNum.value - 1) * pageSize.value
  const end = start + pageSize.value
  return filteredProducts.value.slice(start, end)
})

// 更新总数和页码
watch(filteredProducts, (newVal) => {
  const newTotal = newVal.length
  total.value = newTotal

  // 如果当前页超出总页数，回到第一页
  const maxPage = Math.ceil(newTotal / pageSize.value)
  if (pageNum.value > maxPage && maxPage > 0) {
    pageNum.value = 1
  }
})

// 当筛选条件变化时重置页码
watch([selectedCategoryIds, () => keyword.value, () => orderBy.value], () => {
  if (pageNum.value !== 1) {
    pageNum.value = 1
  }
})

// 加载分类（优化版）
const loadCategories = async () => {
  try {
    const data = await getCategoryTree()
    categories.value = data || []
  } catch (error) {
    console.error('加载分类失败:', error)
    ElMessage.warning('分类加载失败，请刷新重试')
  }
}

// 加载商品（优化版）
const loadProducts = async () => {
  loading.value = true
  try {
    // 后端分页参数（如果后端支持分页和筛选）
    const params: any = {
      pageNum: 1,
      pageSize: 999, // 获取足够多的数据用于前端筛选
    }

    // 如果有分类筛选，可以传给后端减少数据传输
    if (currentCategoryId.value) {
      params.categoryId = currentCategoryId.value
    }
    if (keyword.value) {
      params.keyword = keyword.value
    }

    const res = await getProductList(params)
    productList.value = res.records || []

    // 如果商品数量很大，可以考虑后端分页
    if (productList.value.length >= 500) {
      console.warn('商品数量较多，建议实现后端分页筛选')
    }
  } catch (error) {
    console.error('加载商品失败:', error)
    ElMessage.error('加载商品失败，请稍后重试')
    productList.value = []
  } finally {
    loading.value = false
  }
}

// 选择分类
const selectCategory = (categoryId: string) => {
  if (currentCategoryId.value === categoryId) {
    currentCategoryId.value = ''
    currentCategoryName.value = ''
  } else {
    currentCategoryId.value = categoryId
  }
  // 滚动到顶部
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

// 改变排序
const changeOrder = (type: string) => {
  orderBy.value = type
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

// 搜索
const handleSearch = () => {
  if (keyword.value.trim()) {
    // 可选：记录搜索历史
    console.log('搜索关键词:', keyword.value)
  }
}

// 分页
const handlePageChange = () => {
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

const handleSizeChange = () => {
  pageNum.value = 1
}

// 跳转商品详情
const goDetail = (productId: string) => {
  router.push(`/product/${productId}`)
}

// 监听路由参数变化
watch(() => route.query, (query) => {
  let needReload = false

  if (query.categoryId && query.categoryId !== currentCategoryId.value) {
    currentCategoryId.value = query.categoryId as string
    needReload = true
  }
  if (query.keyword && query.keyword !== keyword.value) {
    keyword.value = query.keyword as string
    needReload = true
  }

  if (needReload) {
    pageNum.value = 1
    loadProducts()
  }
}, { immediate: true, deep: true })

// 初始化
onMounted(async () => {
  await Promise.all([loadCategories(), loadProducts()])
})

// 开发环境调试（可选）
if (import.meta.env.DEV) {
  (window as any).__debug__ = {
    getCategories: () => categories.value,
    getProducts: () => productList.value,
    getCurrentCategoryId: () => currentCategoryId.value,
    getFilteredProducts: () => filteredProducts.value
  }
}
</script>

<style scoped>
.product-list-page {
  background-color: #f8f9fa;
  min-height: calc(100vh - 130px);
  padding: 20px 0;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 20px;
}

.breadcrumb {
  margin-bottom: 20px;
}

.list-wrapper {
  display: flex;
  gap: 30px;
}

/* 侧边栏 */
.sidebar {
  width: 220px;
  flex-shrink: 0;
}

.filter-box {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  position: sticky;
  top: 20px;
}

.filter-box h3 {
  font-size: 16px;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
}

.category-tree {
  display: flex;
  flex-direction: column;
  gap: 4px;
  max-height: calc(100vh - 120px);
  overflow-y: auto;
}

/* 主内容区 */
.main-content {
  flex: 1;
}

/* 搜索栏 */
.search-bar {
  display: flex;
  gap: 12px;
  margin-bottom: 20px;
}

.search-bar .el-input {
  flex: 1;
}

/* 排序栏 */
.sort-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: white;
  padding: 12px 20px;
  border-radius: 8px;
  margin-bottom: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.sort-left {
  display: flex;
  gap: 24px;
}

.sort-right {
  color: #999;
  font-size: 14px;
}

.sort-item {
  cursor: pointer;
  color: #666;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  gap: 4px;
}

.sort-item:hover,
.sort-item.active {
  color: #409eff;
}

.result-count {
  font-size: 14px;
}

/* 商品网格 */
.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
  gap: 20px;
  margin-bottom: 30px;
}

.product-card {
  background: white;
  border-radius: 12px;
  overflow: hidden;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.product-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.15);
}

.product-image {
  position: relative;
  height: 220px;
  overflow: hidden;
  background-color: #f5f5f5;
}

.product-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.3s;
}

.product-card:hover .product-image img {
  transform: scale(1.05);
}

.product-tags {
  position: absolute;
  top: 10px;
  left: 10px;
  display: flex;
  gap: 8px;
}

.tag {
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 500;
}

.tag.hot {
  background: linear-gradient(135deg, #ff6b6b, #ff8e8e);
  color: white;
}

.tag.new {
  background: linear-gradient(135deg, #4facfe, #00f2fe);
  color: white;
}

.product-info {
  padding: 15px;
}

.product-title {
  font-size: 16px;
  margin-bottom: 8px;
  color: #333;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.product-price {
  display: flex;
  align-items: baseline;
  gap: 8px;
  margin-bottom: 8px;
}

.product-price .price {
  color: #ff6b6b;
  font-size: 20px;
  font-weight: bold;
}

.product-price .original-price {
  color: #999;
  font-size: 14px;
  text-decoration: line-through;
}

.product-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
  color: #999;
}

.product-meta .stock.low-stock {
  color: #ff6b6b;
}

/* 分页 */
.pagination {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

/* 响应式 */
@media (max-width: 768px) {
  .list-wrapper {
    flex-direction: column;
  }

  .sidebar {
    width: 100%;
  }

  .filter-box {
    position: static;
  }

  .product-grid {
    grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
    gap: 12px;
  }

  .product-image {
    height: 160px;
  }

  .sort-bar {
    flex-direction: column;
    gap: 10px;
    align-items: flex-start;
  }

  .sort-left {
    flex-wrap: wrap;
    gap: 16px;
  }
}
</style>