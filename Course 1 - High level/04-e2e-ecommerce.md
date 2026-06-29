# React Mastery Guide — Part 4: E2E Example 1
## E-Commerce Platform (Full Architecture)

---

# ARCHITECTURE OVERVIEW

```
┌─────────────────────────────────────────────┐
│                E-Commerce App                │
├─────────────────────────────────────────────┤
│  pages/                                       │
│    HomePage, ProductListing, ProductDetail,   │
│    CartPage, CheckoutPage, OrderHistory       │
├─────────────────────────────────────────────┤
│  features/  (domain modules)                  │
│    products/  catalog/  cart/  checkout/      │
│    orders/  auth/  search/                    │
├─────────────────────────────────────────────┤
│  hooks/  services/  contexts/  components/   │
├─────────────────────────────────────────────┤
│  types/  utils/  constants/                   │
└─────────────────────────────────────────────┘
```

# FULL PROJECT STRUCTURE

```
src/
├── components/
│   ├── ui/           # Button, Input, Modal, Toast
│   ├── layout/       # Header, Footer, Sidebar
│   └── shared/       # ProductCard, Rating, Price
├── features/
│   ├── products/
│   │   ├── components/ ProductCard, ProductGrid, Filters
│   │   ├── hooks/      useProducts, useProduct
│   │   └── api/        productsApi
│   ├── cart/
│   │   ├── components/ CartItem, CartSummary
│   │   ├── hooks/      useCart
│   │   └── store/      cartStore (Zustand)
│   ├── checkout/
│   │   ├── components/ ShippingForm, PaymentForm, Review
│   │   ├── hooks/      useCheckout
│   │   └── api/        checkoutApi
│   ├── orders/
│   │   ├── components/ OrderCard, OrderTimeline
│   │   └── hooks/      useOrders
│   └── auth/
│       ├── components/ LoginForm, ProtectedRoute
│       ├── hooks/      useAuth
│       └── context/    AuthContext
├── services/
│   ├── api.js         # Axios instance, interceptors
│   ├── analytics.js   # Tracking events
│   └── cache.js       # Response caching
├── hooks/
│   ├── useDebounce.js
│   ├── useLocalStorage.js
│   └── useIntersectionObserver.js
├── contexts/
│   ├── AuthContext.jsx
│   └── ThemeContext.jsx
└── utils/
    ├── format.js      # currency, date formatting
    ├── validators.js  # form validation rules
    └── constants.js   # API URLs, config
```

# 1. API LAYER

```jsx
// services/api.js — Axios instance with interceptors
import axios from 'axios';
import { refreshToken, logout } from './auth';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'https://api.example.com',
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
});

// Request interceptor — attach auth token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor — handle token refresh
let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
    if (error) prom.reject(error);
    else prom.resolve(token);
  });
  failedQueue = [];
};

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // If 401 and not already retrying
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue this request until token refreshes
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then(token => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return api(originalRequest);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const { accessToken } = await refreshToken();
        localStorage.setItem('accessToken', accessToken);
        processQueue(null, accessToken);
        originalRequest.headers.Authorization = `Bearer ${accessToken}`;
        return api(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);
        logout();
        window.location.href = '/login';
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);

export default api;
```

# 2. AUTH CONTEXT

```jsx
// contexts/AuthContext.jsx
import { createContext, useContext, useState, useCallback, useEffect } from 'react';
import api from '../services/api';

const AuthContext = createContext(null);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const initAuth = async () => {
      const token = localStorage.getItem('accessToken');
      if (!token) { setLoading(false); return; }
      try {
        const { data } = await api.get('/auth/me');
        setUser(data.user);
      } catch {
        localStorage.removeItem('accessToken');
      } finally {
        setLoading(false);
      }
    };
    initAuth();
  }, []);

  const login = useCallback(async (email, password) => {
    const { data } = await api.post('/auth/login', { email, password });
    localStorage.setItem('accessToken', data.accessToken);
    localStorage.setItem('refreshToken', data.refreshToken);
    setUser(data.user);
    return data.user;
  }, []);

  const register = useCallback(async (userData) => {
    const { data } = await api.post('/auth/register', userData);
    localStorage.setItem('accessToken', data.accessToken);
    setUser(data.user);
    return data.user;
  }, []);

  const logout = useCallback(async () => {
    try { await api.post('/auth/logout'); } catch {}
    localStorage.removeItem('accessToken');
    localStorage.removeItem('refreshToken');
    setUser(null);
  }, []);

  const updateProfile = useCallback(async (profileData) => {
    const { data } = await api.put('/auth/profile', profileData);
    setUser(data.user);
    return data.user;
  }, []);

  const value = useMemo(() => ({
    user, loading, login, register, logout, updateProfile,
    isAuthenticated: !!user,
    isAdmin: user?.role === 'admin',
  }), [user, loading, login, register, logout, updateProfile]);

  return (
    <AuthContext.Provider value={value}>
      {!loading ? children : <FullPageSpinner />}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
};
```

# 3. CART STORE (Zustand)

```jsx
// features/cart/store/cartStore.js
import { create } from 'zustand';
import { persist, devtools } from 'zustand/middleware';
import api from '../../../services/api';

export const useCartStore = create(
  devtools(
    persist(
      (set, get) => ({
        // State
        items: [],
        coupon: null,
        shippingMethod: 'standard',
        isSyncing: false,

        // Computed (via getters)
        getTotal: () => {
          const items = get().items;
          const subtotal = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
          const discount = get().getDiscount();
          const shipping = get().shippingMethod === 'express' ? 15 : 5;
          return { subtotal, discount, shipping, total: subtotal - discount + shipping };
        },

        getItemCount: () => get().items.reduce((sum, i) => sum + i.quantity, 0),

        getDiscount: () => {
          const coupon = get().coupon;
          if (!coupon) return 0;
          const subtotal = get().items.reduce((sum, i) => sum + i.price * i.quantity, 0);
          return coupon.type === 'percentage' ? subtotal * (coupon.value / 100) : coupon.value;
        },

        // Actions
        addItem: (product, quantity = 1) =>
          set((state) => {
            const existing = state.items.find(i => i.id === product.id);
            const newItems = existing
              ? state.items.map(i =>
                  i.id === product.id ? { ...i, quantity: i.quantity + quantity } : i
                )
              : [...state.items, { ...product, quantity }];
            return { items: newItems };
          }),

        removeItem: (productId) =>
          set((state) => ({ items: state.items.filter(i => i.id !== productId) })),

        updateQuantity: (productId, quantity) =>
          set((state) => ({
            items: quantity <= 0
              ? state.items.filter(i => i.id !== productId)
              : state.items.map(i => i.id === productId ? { ...i, quantity } : i),
          })),

        applyCoupon: async (code) => {
          try {
            const { data } = await api.post('/cart/validate-coupon', { code });
            set({ coupon: data });
            return { success: true, message: data.message };
          } catch (error) {
            return { success: false, message: error.response?.data?.message || 'Invalid coupon' };
          }
        },

        removeCoupon: () => set({ coupon: null }),

        setShippingMethod: (method) => set({ shippingMethod: method }),

        syncCart: async () => {
          set({ isSyncing: true });
          try {
            await api.post('/cart/sync', { items: get().items });
          } catch {} finally {
            set({ isSyncing: false });
          }
        },

        clearCart: () => set({ items: [], coupon: null, shippingMethod: 'standard' }),
      }),
      {
        name: 'shopping-cart',
        partialize: (state) => ({ items: state.items }),
      }
    ),
    { name: 'CartStore' }
  )
);
```

# 4. PRODUCT LISTING WITH FILTERS

```jsx
// features/products/hooks/useProducts.js
import { useState, useEffect, useCallback, useMemo } from 'react';
import { useSearchParams } from 'react-router-dom';
import api from '../../../services/api';
import { useDebounce } from '../../../hooks/useDebounce';

const PAGE_SIZE = 20;

export const useProducts = () => {
  const [searchParams, setSearchParams] = useSearchParams();

  // Sync state with URL params
  const [filters, setFilters] = useState({
    category: searchParams.get('category') || '',
    minPrice: searchParams.get('minPrice') || '',
    maxPrice: searchParams.get('maxPrice') || '',
    rating: searchParams.get('rating') || '',
    sortBy: searchParams.get('sortBy') || 'newest',
    inStock: searchParams.get('inStock') === 'true',
  });

  const [page, setPage] = useState(Number(searchParams.get('page')) || 1);
  const [searchTerm, setSearchTerm] = useState(searchParams.get('q') || '');
  const debouncedSearch = useDebounce(searchTerm, 400);

  const [products, setProducts] = useState([]);
  const [totalProducts, setTotalProducts] = useState(0);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  // Update URL when filters change
  useEffect(() => {
    const params = new URLSearchParams();
    if (filters.category) params.set('category', filters.category);
    if (filters.minPrice) params.set('minPrice', filters.minPrice);
    if (filters.maxPrice) params.set('maxPrice', filters.maxPrice);
    if (filters.rating) params.set('rating', filters.rating);
    if (filters.sortBy && filters.sortBy !== 'newest') params.set('sortBy', filters.sortBy);
    if (filters.inStock) params.set('inStock', 'true');
    if (debouncedSearch) params.set('q', debouncedSearch);
    if (page > 1) params.set('page', String(page));
    setSearchParams(params, { replace: true });
  }, [filters, debouncedSearch, page, setSearchParams]);

  // Fetch products
  useEffect(() => {
    let cancelled = false;
    const controller = new AbortController();

    const fetchProducts = async () => {
      setLoading(true);
      setError(null);
      try {
        const params = new URLSearchParams({
          page: String(page),
          limit: String(PAGE_SIZE),
          ...(filters.category && { category: filters.category }),
          ...(filters.minPrice && { minPrice: filters.minPrice }),
          ...(filters.maxPrice && { maxPrice: filters.maxPrice }),
          ...(filters.rating && { minRating: filters.rating }),
          ...(filters.sortBy && { sortBy: filters.sortBy }),
          ...(filters.inStock && { inStock: 'true' }),
          ...(debouncedSearch && { q: debouncedSearch }),
        });

        const { data } = await api.get(`/products?${params}`, {
          signal: controller.signal,
        });

        if (!cancelled) {
          setProducts(data.products);
          setTotalProducts(data.total);
        }
      } catch (err) {
        if (!cancelled && err.name !== 'AbortError') {
          setError(err.response?.data?.message || 'Failed to load products');
        }
      } finally {
        if (!cancelled) setLoading(false);
      }
    };

    fetchProducts();
    return () => { cancelled = true; controller.abort(); };
  }, [filters, debouncedSearch, page]);

  const totalPages = Math.ceil(totalProducts / PAGE_SIZE);

  const updateFilter = useCallback((key, value) => {
    setFilters(prev => ({ ...prev, [key]: value }));
    setPage(1);
  }, []);

  const clearFilters = useCallback(() => {
    setFilters({ category: '', minPrice: '', maxPrice: '', rating: '', sortBy: 'newest', inStock: false });
    setSearchTerm('');
    setPage(1);
  }, []);

  const activeFilterCount = useMemo(() =>
    Object.entries(filters).filter(([k, v]) => v && k !== 'sortBy').length,
    [filters]
  );

  return {
    products, loading, error, totalProducts,
    page, setPage, totalPages,
    filters, updateFilter, clearFilters, activeFilterCount,
    searchTerm, setSearchTerm,
  };
};
```

# 5. PRODUCT COMPONENT

```jsx
// features/products/components/ProductCard.jsx
import { memo, useState, useCallback } from 'react';
import { Link } from 'react-router-dom';
import { useCartStore } from '../../cart/store/cartStore';
import { formatPrice } from '../../../utils/format';

const IMAGE_SIZES = '(max-width: 768px) 50vw, (max-width: 1200px) 33vw, 25vw';

export const ProductCard = memo(({ product, isPriority = false }) => {
  const [imageLoaded, setImageLoaded] = useState(false);
  const [imageError, setImageError] = useState(false);
  const addItem = useCartStore(state => state.addItem);
  const [added, setAdded] = useState(false);

  const handleAddToCart = useCallback((e) => {
    e.preventDefault(); // Prevent navigating to product detail
    e.stopPropagation();
    addItem(product);
    setAdded(true);
    setTimeout(() => setAdded(false), 2000);
  }, [product, addItem]);

  const discount = product.compareAtPrice
    ? Math.round((1 - product.price / product.compareAtPrice) * 100)
    : 0;

  return (
    <div className="product-card">
      <Link to={`/products/${product.slug}`} className="product-card__link">
        <div className="product-card__image-wrapper">
          {!imageLoaded && !imageError && <Skeleton className="product-card__skeleton" />}
          {imageError ? (
            <div className="product-card__image-fallback">📷</div>
          ) : (
            <img
              src={product.images[0]}
              srcSet={product.images.map((img, i) =>
                `${img} ${i === 0 ? '1x' : '2x'}`
              ).join(', ')}
              sizes={IMAGE_SIZES}
              alt={product.name}
              loading={isPriority ? 'eager' : 'lazy'}
              onLoad={() => setImageLoaded(true)}
              onError={() => setImageError(true)}
              className={`product-card__image ${imageLoaded ? 'loaded' : ''}`}
            />
          )}

          {discount > 0 && (
            <span className="product-card__badge">-{discount}%</span>
          )}

          {!product.inStock && (
            <div className="product-card__out-of-stock">Out of Stock</div>
          )}
        </div>

        <div className="product-card__info">
          <h3 className="product-card__name">{product.name}</h3>
          <div className="product-card__rating">
            <Stars rating={product.rating} />
            <span>({product.reviewCount})</span>
          </div>
          <div className="product-card__pricing">
            <span className="product-card__price">{formatPrice(product.price)}</span>
            {product.compareAtPrice && (
              <span className="product-card__compare-price">{formatPrice(product.compareAtPrice)}</span>
            )}
          </div>
        </div>
      </Link>

      <button
        className={`product-card__add-btn ${added ? 'added' : ''}`}
        onClick={handleAddToCart}
        disabled={!product.inStock}
      >
        {added ? '✓ Added!' : product.inStock ? 'Add to Cart' : 'Sold Out'}
      </button>
    </div>
  );
});

ProductCard.displayName = 'ProductCard';
```

# 6. CHECKOUT FLOW

```jsx
// features/checkout/hooks/useCheckout.js
import { useState, useCallback, useReducer } from 'react';
import { useNavigate } from 'react-router-dom';
import api from '../../../services/api';
import { useCartStore } from '../../cart/store/cartStore';

const initialState = {
  shipping: {
    firstName: '', lastName: '', address: '', city: '',
    state: '', zip: '', country: 'US', phone: '',
  },
  payment: {
    cardNumber: '', expMonth: '', expYear: '', cvv: '', nameOnCard: '',
    saveCard: false,
  },
  billing: {
    sameAsShipping: true,
    firstName: '', lastName: '', address: '', city: '',
    state: '', zip: '', country: 'US',
  },
  errors: {},
  step: 1,
  isProcessing: false,
  orderError: null,
};

function checkoutReducer(state, action) {
  switch (action.type) {
    case 'UPDATE_SHIPPING':
      return { ...state, shipping: { ...state.shipping, ...action.payload } };
    case 'UPDATE_PAYMENT':
      return { ...state, payment: { ...state.payment, ...action.payload } };
    case 'UPDATE_BILLING':
      return { ...state, billing: { ...state.billing, ...action.payload } };
    case 'SET_SAME_BILLING':
      return { ...state, billing: { ...state.billing, sameAsShipping: action.payload } };
    case 'SET_ERRORS':
      return { ...state, errors: action.payload };
    case 'NEXT_STEP':
      return { ...state, step: Math.min(state.step + 1, 4) };
    case 'PREV_STEP':
      return { ...state, step: Math.max(state.step - 1, 1), errors: {} };
    case 'PROCESSING':
      return { ...state, isProcessing: action.payload, orderError: null };
    case 'ERROR':
      return { ...state, isProcessing: false, orderError: action.payload };
    default:
      return state;
  }
}

export const useCheckout = () => {
  const [state, dispatch] = useReducer(checkoutReducer, initialState);
  const { items, total, clearCart } = useCartStore();
  const navigate = useNavigate();

  const validateStep = useCallback((step) => {
    const errors = {};

    if (step === 1) {
      const { shipping } = state;
      if (!shipping.firstName.trim()) errors.firstName = 'Required';
      if (!shipping.lastName.trim()) errors.lastName = 'Required';
      if (!shipping.address.trim()) errors.address = 'Required';
      if (!shipping.city.trim()) errors.city = 'Required';
      if (!shipping.zip.trim()) errors.zip = 'Required';
      if (!shipping.phone.trim()) errors.phone = 'Required';
    }

    if (step === 2) {
      const { payment } = state;
      if (!payment.cardNumber.replace(/\s/g, '').match(/^\d{16}$/)) errors.cardNumber = 'Invalid card';
      if (!payment.expMonth || !payment.expYear) errors.expiry = 'Required';
      if (!payment.cvv.match(/^\d{3,4}$/)) errors.cvv = 'Invalid CVV';
      if (!payment.nameOnCard.trim()) errors.nameOnCard = 'Required';
    }

    dispatch({ type: 'SET_ERRORS', payload: errors });
    return Object.keys(errors).length === 0;
  }, [state]);

  const nextStep = useCallback(() => {
    if (validateStep(state.step)) {
      dispatch({ type: 'NEXT_STEP' });
    }
  }, [state.step, validateStep]);

  const prevStep = useCallback(() => dispatch({ type: 'PREV_STEP' }), []);

  const placeOrder = useCallback(async () => {
    dispatch({ type: 'PROCESSING', payload: true });
    try {
      const orderData = {
        items,
        shipping: state.shipping,
        payment: {
          lastFour: state.payment.cardNumber.slice(-4),
          cardType: detectCardType(state.payment.cardNumber),
        },
        billing: state.billing.sameAsShipping ? state.shipping : state.billing,
      };

      const { data } = await api.post('/orders', orderData);
      clearCart();
      navigate(`/order-confirmation/${data.orderId}`);
      return data;
    } catch (error) {
      dispatch({
        type: 'ERROR',
        payload: error.response?.data?.message || 'Payment failed. Please try again.',
      });
      throw error;
    }
  }, [items, state, clearCart, navigate]);

  return { state, dispatch, nextStep, prevStep, placeOrder };
};
```

# 7. COMPLETE CHECKOUT PAGE

```jsx
// pages/CheckoutPage.jsx
import { useCheckout } from '../features/checkout/hooks/useCheckout';
import { useCartStore } from '../features/cart/store/cartStore';
import { ShippingForm } from '../features/checkout/components/ShippingForm';
import { PaymentForm } from '../features/checkout/components/PaymentForm';
import { OrderReview } from '../features/checkout/components/OrderReview';
import { formatPrice } from '../utils/format';

const STEPS = ['Shipping', 'Payment', 'Review', 'Confirmation'];

export const CheckoutPage = () => {
  const { state, nextStep, prevStep, placeOrder } = useCheckout();
  const { items, getTotal } = useCartStore();
  const { subtotal, discount, shipping, total } = getTotal();

  if (!items.length) return <EmptyCart />;

  return (
    <div className="checkout-page">
      <div className="checkout-page__main">
        <CheckoutProgress currentStep={state.step} steps={STEPS} />

        {state.step === 1 && (
          <ShippingForm
            data={state.shipping}
            errors={state.errors}
            onChange={(data) => dispatch({ type: 'UPDATE_SHIPPING', payload: data })}
            onNext={nextStep}
          />
        )}

        {state.step === 2 && (
          <PaymentForm
            data={state.payment}
            errors={state.errors}
            onChange={(data) => dispatch({ type: 'UPDATE_PAYMENT', payload: data })}
            onNext={nextStep}
            onBack={prevStep}
            sameAsShipping={state.billing.sameAsShipping}
            onToggleBilling={() => dispatch({ type: 'SET_SAME_BILLING', payload: !state.billing.sameAsShipping })}
          />
        )}

        {state.step === 3 && (
          <OrderReview
            shipping={state.shipping}
            payment={state.payment}
            billing={state.billing}
            onBack={prevStep}
            onPlaceOrder={placeOrder}
            isProcessing={state.isProcessing}
            error={state.orderError}
          />
        )}

        {state.step === 4 && <OrderConfirmation />}
      </div>

      <aside className="checkout-page__sidebar">
        <OrderSummary
          items={items}
          subtotal={subtotal}
          discount={discount}
          shipping={shipping}
          total={total}
        />
      </aside>
    </div>
  );
};
```

# INTERVIEW TIPS FOR E-COMMERCE

1. **Race conditions** — Always mention abort controllers and `cancelled` flag for fetch requests
2. **Optimistic updates** — Show how cart quantity changes update instantly, rollback on error
3. **State normalization** — Normalized state prevents duplicate data and simplifies updates
4. **Checkout integrity** — Never trust client-side validation alone; always re-validate server-side
5. **Payment security** — Never store full card numbers client-side; use PCI-compliant payment gateway
6. **Performance** — Virtualize product lists, lazy load images, preload critical chunks
7. **Error recovery** — Handle network failures, show meaningful errors, retry mechanisms
8. **Analytics** — Tracking every step of funnel (view product → add to cart → checkout → purchase)

---

# 5 PRO INTERVIEW QUESTIONS

**Q1:** How would you handle race conditions when a user rapidly changes product filters?
**A:** Use AbortController to cancel previous in-flight requests. Track a `requestId` counter — only apply response if it matches latest request. Alternatively, use debounce on filter changes.

**Q2:** How do you prevent users from navigating away during checkout with unsaved data?
**A:** Use `beforeunload` event listener via useEffect. For SPA navigation, use `useBlocker` (React Router v6) or a custom history blocker. Show a confirmation dialog.

**Q3:** How would you implement real-time inventory updates when two users buy the same item?
**A:** Use WebSocket connection for live inventory. When stock changes, push event from server. Use optimistic UI with rollback — decrement stock locally, if server rejects, restore. Show "Only N left" urgency messaging.

**Q4:** How do you handle cart persistence across devices?
**A:** Sync cart to server (authenticated users) via API calls. Use localStorage for anonymous users with a `cartToken` server-side. On login, merge anonymous cart with server cart. Use Zustand's persist middleware for client persistence.

**Q5:** Design a product recommendation carousel component that's performant with 1000+ products.
**A:** Lazy load images, virtualize visible items (react-window horizontal list), preload adjacent items, use CSS transforms for smooth scrolling, debounce scroll events, implement intersection observer for analytics tracking, and use React.memo for each item.
