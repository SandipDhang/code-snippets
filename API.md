import axios, {
  AxiosError,
  AxiosResponse,
  InternalAxiosRequestConfig,
} from 'axios';

import { store } from '@/store/store';
import { logout } from '@/store/reducers/userSlice';

// Get the API base URL from the environment variables
const baseURL = import.meta.env.HAI_APP_API;

const axiosInstance = axios.create({ baseURL });

// Extend InternalAxiosRequestConfig to include a custom `_retry` field
interface CustomAxiosRequestConfig extends InternalAxiosRequestConfig {
  _retry?: boolean;
}

// Array of API paths that should NOT include the Bearer token
const excludeBearerTokenPaths = ['/login', '/register', '/public-route'];

// Maintain a map to track pending API requests
const pendingRequests: Map<string, AbortController> = new Map();

// Helper function to check if a URL should exclude the Bearer token
const shouldExcludeBearerToken = (url: string) => {
  return excludeBearerTokenPaths.some((path) => url.includes(path));
};

// Add the request interceptor
axiosInstance.interceptors.request.use(
  (config: CustomAxiosRequestConfig) => {
    const token = localStorage.getItem('HAI_AUTH_TOKEN');

    // Add Bearer token if the API is not in the excluded list
    if (token && !shouldExcludeBearerToken(config.url || '')) {
      // Use AxiosHeaders to set headers
      if (config.headers && 'set' in config.headers) {
        config.headers.set('Authorization', `Bearer ${token}`);
      }
    }

    // Handle duplicate API calls (cancel the previous one if it's still pending)
    const requestKey = `${config.method}-${config.url}`;
    if (pendingRequests.has(requestKey)) {
      const controller = pendingRequests.get(requestKey);
      if (controller) controller.abort(); // Cancel previous request
      pendingRequests.delete(requestKey);
    }

    // Add AbortController to cancel duplicate requests
    const controller = new AbortController();
    config.signal = controller.signal;
    pendingRequests.set(requestKey, controller);

    return config;
  },
  (error: AxiosError) => {
    return Promise.reject(error);
  }
);

// Add the response interceptor
axiosInstance.interceptors.response.use(
  (response: AxiosResponse) => {
    // Clear the pending request after the response is received
    const requestKey = `${response.config.method}-${response.config.url}`;
    pendingRequests.delete(requestKey);

    return response;
  },
  async (error: AxiosError) => {
    const originalRequest = error.config as CustomAxiosRequestConfig;
    const requestKey = `${originalRequest.method}-${originalRequest.url}`;

    // Token expired, try to refresh it
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      // Try to refresh the token
      const refreshToken = localStorage.getItem('RUS_AUTH_REF_TOKEN');
      if (refreshToken) {
        try {
          const response = await axios.post(`${baseURL}/refresh-token`, {
            refresh_token: refreshToken,
          });

          const { access_token } = response.data;
          localStorage.setItem('HAI_AUTH_TOKEN', access_token);

          // Update the Authorization header and retry the original request
          if (originalRequest.headers && 'set' in originalRequest.headers) {
            originalRequest.headers.set(
              'Authorization',
              `Bearer ${access_token}`
            );
          }

          return axiosInstance(originalRequest); // Retry the request with the new token
        } catch (refreshError) {
          console.log(refreshError);
          // Refresh token failed, redirect to login
          store.dispatch(logout());
          window.location.href = '/login';
        }
      } else {
        // No refresh token, redirect to login
        store.dispatch(logout());
        window.location.href = '/login';
      }
    }

    pendingRequests.delete(requestKey);
    return Promise.reject(error);
  }
);

export default axiosInstance;
