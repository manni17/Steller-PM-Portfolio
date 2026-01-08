# Steller Frontend Dashboards Documentation

This document provides comprehensive information about the Steller frontend dashboards - both the admin dashboard and consumer dashboard built with Vue.js.

## Overview

The Steller platform includes two Vue.js 2.6 dashboards built from the same Vuexy template:

1. **Steller Admin Dashboard**: For administrative functions (connects to Steller API at port 5091)
2. **Steller Consumer Dashboard**: For partner/consumer functions (connects to StellerConsumer API at port 5092)

Both dashboards share the same underlying architecture but serve different purposes and connect to different backend services.

## Architecture and Structure

### Technology Stack
- **Framework**: Vue.js 2.6.12 with Vue CLI
- **UI Library**: BootstrapVue 2.21.1 with Vuexy template
- **State Management**: Vuex 3.6.0
- **Routing**: Vue Router 3.4.9
- **HTTP Client**: Axios with custom interceptors
- **Form Validation**: VeeValidate 3.4.5
- **Icons**: Feather Icons
- **Styling**: SCSS with Bootstrap 4.6.0

### Project Structure
```
steller-{admin|consumer}-dashboard/
├── public/                 # Static assets
├── src/
│   ├── @core/             # Core framework components
│   ├── assets/            # Images, icons, styles
│   ├── auth/              # Authentication utilities
│   ├── layouts/           # Page layouts
│   ├── libs/              # Third-party library integrations
│   ├── navigation/        # Menu navigation
│   ├── router/            # Vue Router configuration
│   ├── store/             # Vuex state management
│   ├── utils/             # Utility functions
│   ├── views/             # Page components
│   ├── App.vue            # Root component
│   ├── main.js            # Application entry point
│   └── global-components.js # Globally registered components
├── package.json           # Dependencies and scripts
├── vue.config.js          # Vue CLI configuration
├── Dockerfile             # Container configuration
└── nginx.conf             # Production web server config
```

## Dashboard Differences

### Admin Dashboard (Steller)
- **API Endpoint**: http://148.135.138.209:5091/api (production) or http://localhost:5091/api
- **App Name**: "Steller"
- **Purpose**: Administrative functions and system management
- **Key Features**: Partner management, user administration, system reporting

### Consumer Dashboard (StellerConsumer)
- **API Endpoint**: http://localhost:5092/api
- **App Name**: "StellerConsumer"
- **Purpose**: Partner/consumer functions and order placement
- **Key Features**: Product browsing, cart management, order placement

## Key Features and Functionality

### Authentication System
- **Login Flow**: `/Account/Login` API with JWT token response
- **Token Storage**: Secure cookies with app-specific naming (e.g., "Steller_AUTH_TOKEN")
- **Route Protection**: Router middleware checks for authentication tokens
- **Auto Redirect**: Unauthenticated users redirected to login
- **Axios Integration**: Interceptors automatically attach Authorization headers

### Admin Dashboard Features
1. **Partner Management**: Full CRUD operations for business partners
2. **User Administration**: Manage admin users for each partner
3. **Catalog Management**: Browse and manage product catalogs
4. **Order Reporting**: Track and report on orders with detailed views
5. **Wallet Management**: Handle wallets and transaction history
6. **System Administration**: General system management functions

### Consumer Dashboard Features
1. **Product Catalog**: Browse products with search and filtering
2. **Shopping Cart**: Add/remove items with real-time summary
3. **Order Placement**: Multi-step order creation with API key authentication
4. **Order Tracking**: View order status and history

### Special Consumer Feature: Make Order Flow
1. **Product Browsing**: Grid-based catalog with search and pagination
2. **Cart Management**: Add/remove items with quantity controls
3. **API Key Retrieval**: Automatic fetching of partner API key
4. **Order Placement**: Submit cart with auto-generated UUID and API key header
5. **Success Handling**: Clear cart after successful order submission

## API Integration

### Base Configuration
```javascript
// src/libs/axios.js
axios.defaults.baseURL = process.env.VUE_APP_URL;

axios.interceptors.request.use(function(config) {
  const AUTH_TOKEN = $cookies.get(process.env.VUE_APP_NAME + '_AUTH_TOKEN')
  if (AUTH_TOKEN) {
      config.headers.common['Authorization'] = AUTH_TOKEN
  }
  return config
})
```

### Common API Endpoints
- **Authentication**: `/Account/Login`
- **Partners**: `/Partner`, `/Partner/:id`
- **Users**: `/users/UpdateUserStatus`
- **Products**: `/Product/partnerBrands` (consumer), `/Product` (admin)
- **Orders**: `/Order` (consumer), order reporting endpoints (admin)
- **Wallets**: `/Wallet/create`, `/Currency`

### Consumer-Specific Integration
- **API Key System**: `/Partner/PartnerApiKey` for secure partner operations
- **Order Flow**: POST to `/Order` with X-Api-Key header and RequestId
- **UUID Generation**: Client-side UUID generation for request tracking

## Security Implementation

### Authentication Security
- **JWT Storage**: Secure cookie storage (better than localStorage)
- **Route Protection**: Router-level authentication checks
- **Axios Interceptors**: Automatic token attachment to requests
- **Token Refresh**: Not implemented, potential area for improvement

### API Key Security
- **Dynamic Retrieval**: API keys fetched as needed for partner operations
- **Header Attachment**: Automatic X-Api-Key header for partner-specific requests
- **Partner Isolation**: Each partner's operations are secured with their own key

### Potential Security Improvements
- **Cookie Security**: Add secure flag and SameSite attributes
- **Token Management**: Implement token refresh mechanism
- **CSRF Protection**: Add CSRF tokens for additional security
- **Input Sanitization**: Add client-side sanitization where appropriate

## UI/UX Components and Patterns

### Layout System
- **Responsive Grid**: Bootstrap 4 responsive grid system
- **Card-Based Layout**: Content organized in Bootstrap cards
- **Sidebar Navigation**: Collapsible menu with icons
- **Breadcrumbs**: Hierarchical navigation indicators

### Component Library
- **Data Tables**: Feature-rich data tables with sorting, filtering, pagination
- **Forms**: Comprehensive form components with validation
- **Charts**: Multiple chart libraries (ApexCharts, Chart.js, ECharts)
- **Modals**: Confirmation dialogs and data entry forms
- **Notifications**: Toast notifications for user feedback

### User Experience Features
- **Loading States**: Overlay spinners during async operations
- **Empty States**: Illustrations and messaging for empty data
- **Search Functionality**: Debounced search with results
- **Pagination**: Server-side pagination controls
- **Responsive Design**: Mobile-first responsive approach

## Build and Deployment

### Build Configuration
- **Vue CLI**: 4.5.x with optimized build settings
- **Asset Optimization**: Minification and compression enabled
- **Code Splitting**: Route-based and component-based chunking
- **Production Settings**: Source maps disabled for production

### Deployment Configuration
- **Docker**: Multi-stage Docker builds for production
- **Nginx**: Production serving with SPA configuration
- **Environment Variables**: Runtime configuration via .env files
- **Reverse Proxy**: Proper proxying to backend APIs

### Environment Configuration
- **Development**: Local API endpoints
- **Production**: Remote API endpoints (admin dashboard uses production server)

## Performance Considerations

### Current Optimizations
- **Component Code Splitting**: Lazy loading for better initial load
- **Image Optimization**: Proper loading and error handling
- **API Pagination**: Server-side pagination for large datasets
- **Axios Interceptors**: Efficient request/response handling

### Areas for Improvement
- **Bundle Analysis**: Implement bundle size monitoring
- **Caching Strategy**: Add request caching for read operations
- **Virtual Scrolling**: Implement for large data sets
- **Offline Support**: Add service worker for offline capability

## Development Patterns

### Service Layer Pattern
Each feature has a dedicated service class for API calls:
```javascript
class DataService {
  getAll(page, query) {
    // Standardized API call pattern
  }
}
```

### Component Architecture
- **Page Components**: In views/ directory with specific functionality
- **Reusable Components**: In @core/ and shared locations
- **Form Components**: With integrated validation
- **Layout Components**: For consistent UI structure

### State Management
- **Vuex Store**: Centralized state management
- **Modules**: Organized by feature
- **Actions/Getters**: Proper separation of data operations

## Internationalization
- **Arabic Support**: Default localization configuration (`localize('ar')`)
- **Moment.js**: For date/time formatting
- **Vue-i18n**: For text translation (though not heavily utilized)

## Monitoring and Error Handling

### Error Handling
- **API Error Handling**: Proper error status and message handling
- **User Notifications**: Toast notifications for error feedback
- **Graceful Degradation**: UI handles API failures gracefully

### Logging
- **Console Logging**: For development debugging
- **User Feedback**: Clear error messages through UI notifications
- **Request/Response**: Interceptor-based request response logging

## Best Practices Implemented

1. **Code Organization**: Clear separation of concerns with dedicated directories
2. **Component Design**: Reusable, focused components
3. **API Integration**: Standardized service layer pattern
4. **Authentication**: Secure token handling with interceptors
5. **Routing**: Proper route protection and navigation
6. **Error Handling**: Comprehensive error handling and user feedback
7. **Responsive Design**: Mobile-first approach with Bootstrap
8. **SASS Integration**: Modular styling with Bootstrap override capability

## Recommendations for Improvement

### Security
- Implement secure cookie flags (secure, SameSite)
- Add token refresh mechanism
- Implement proper CSP headers
- Add input sanitization

### Performance
- Implement bundle analysis
- Add request caching
- Optimize images and assets
- Implement virtual scrolling for large lists

### Code Quality
- Migrate to TypeScript
- Add comprehensive unit tests
- Implement composition API
- Add automated linting and formatting

### UX
- Add skeleton screens during loading
- Improve accessibility compliance
- Add keyboard navigation
- Implement better offline support

### Monitoring
- Add client-side error tracking
- Implement performance monitoring
- Add usage analytics
- Add audit logging for important actions