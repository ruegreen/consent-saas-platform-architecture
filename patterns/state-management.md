# Pattern: State Management (React Context + TypeScript)

## Modern React Context Architecture

The multi-brand consent platform uses React Context with TypeScript for state management across both the admin interface and public forms. The state architecture is brand-aware, with each context providing brand-scoped data and operations.

### Context Structure

```typescript
// Brand-aware context providers
import { createContext, useContext } from 'react';

// Brand Context - provides current brand configuration
interface BrandContextType {
  brand: Brand | null;
  isLoading: boolean;
  switchBrand: (brandSlug: string) => Promise<void>;
  updateBrandTheme: (theme: BrandTheme) => void;
}

// Auth Context - handles brand-scoped authentication
interface AuthContextType {
  user: AuthUser | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  checkAuth: () => Promise<void>;
}

// Admin Data Context - brand-scoped admin operations
interface AdminContextType {
  tenants: Tenant[];
  providers: Provider[];
  consentForms: ConsentForm[];
  submissions: Submission[];
  isLoading: Record<string, boolean>;
  error: Record<string, string | null>;
}
```

### Brand-Aware State Shape

```typescript
// Platform state automatically scoped to current brand
interface PlatformState {
  // Brand context (resolved from hostname)
  currentBrand: {
    brand: Brand;
    theme: BrandTheme;
    settings: BrandSettings;
  };

  // Authentication (brand-scoped)
  auth: {
    user: AdminUser | null;
    session: Session | null;
    permissions: UserPermissions;
  };

  // Brand entities (automatically filtered by brandId)
  entities: {
    tenants: Tenant[];           // Locations within brand
    providers: Provider[];       // Staff/artists/doctors within brand
    consentForms: ConsentForm[]; // Forms available to brand
    clauses: Clause[];          // Legal clauses for brand
    submissions: Submission[];   // Consent submissions for brand
    adminUsers: AdminUser[];    // Users who can manage brand
  };

  // UI state
  ui: {
    loading: Record<string, boolean>;
    errors: Record<string, string | null>;
    notifications: Notification[];
  };
}
```

### API Integration Pattern (Brand-Aware)

Every API call is automatically brand-scoped through the brand context:

```typescript
// Custom hook for brand-aware API calls
function useAdminApi() {
  const { brand } = useBrand();
  const { user } = useAuth();

  // All API calls include brand context automatically
  const apiClient = useMemo(() => {
    return new ApiClient({
      baseURL: '/api/v1',
      brandId: brand?._id,
      headers: {
        'Authorization': `Bearer ${user?.token}`,
        'X-Brand-Context': brand?.slug
      }
    });
  }, [brand, user]);

  return {
    // Tenant operations (brand-scoped)
    tenants: {
      list: () => apiClient.get('/admin/tenants'),
      create: (tenant: CreateTenantRequest) => 
        apiClient.post('/admin/tenants', tenant),
      update: (id: string, tenant: UpdateTenantRequest) => 
        apiClient.patch(`/admin/tenants/${id}`, tenant),
      delete: (id: string) => 
        apiClient.delete(`/admin/tenants/${id}`)
    },

    // Provider operations (brand-scoped)
    providers: {
      list: () => apiClient.get('/admin/providers'),
      create: (provider: CreateProviderRequest) => 
        apiClient.post('/admin/providers', provider),
      update: (id: string, provider: UpdateProviderRequest) => 
        apiClient.patch(`/admin/providers/${id}`, provider),
      delete: (id: string) => 
        apiClient.delete(`/admin/providers/${id}`)
    },

    // Consent form operations (brand-scoped)
    consentForms: {
      list: () => apiClient.get('/admin/consent-forms'),
      create: (form: CreateFormRequest) => 
        apiClient.post('/admin/consent-forms', form),
      update: (id: string, form: UpdateFormRequest) => 
        apiClient.patch(`/admin/consent-forms/${id}`, form),
      publish: (id: string) => 
        apiClient.post(`/admin/consent-forms/${id}/publish`),
      unpublish: (id: string) => 
        apiClient.post(`/admin/consent-forms/${id}/unpublish`)
    },

    // Submission operations (brand-scoped)
    submissions: {
      list: (filters?: SubmissionFilters) => 
        apiClient.get('/admin/submissions', { params: filters }),
      get: (id: string) => 
        apiClient.get(`/admin/submissions/${id}`),
      update: (id: string, submission: UpdateSubmissionRequest) => 
        apiClient.patch(`/admin/submissions/${id}`, submission)
    }
  };
}
```

### Custom Hooks Pattern

```typescript
// useResource hook - generic pattern for CRUD operations
function useResource<T>(
  resource: string,
  initialData: T[] = []
) {
  const [data, setData] = useState<T[]>(initialData);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const api = useAdminApi();

  const load = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const response = await api[resource].list();
      setData(response.data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [api, resource]);

  const create = useCallback(async (item: Partial<T>) => {
    setLoading(true);
    try {
      const response = await api[resource].create(item);
      setData(prev => [...prev, response.data]);
      return response.data;
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  }, [api, resource]);

  const update = useCallback(async (id: string, item: Partial<T>) => {
    setLoading(true);
    try {
      const response = await api[resource].update(id, item);
      setData(prev => prev.map(x => 
        x.id === id ? { ...x, ...response.data } : x
      ));
      return response.data;
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  }, [api, resource]);

  const remove = useCallback(async (id: string) => {
    setLoading(true);
    try {
      await api[resource].delete(id);
      setData(prev => prev.filter(x => x.id !== id));
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  }, [api, resource]);

  return {
    data,
    loading,
    error,
    load,
    create,
    update,
    remove,
    refresh: load
  };
}

// Usage in components
function TenantsPage() {
  const {
    data: tenants,
    loading,
    error,
    load,
    create,
    update,
    remove
  } = useResource<Tenant>('tenants');

  useEffect(() => {
    load();
  }, [load]);

  return (
    <div>
      {loading && <LoadingSpinner />}
      {error && <ErrorAlert message={error} />}
      <TenantsList
        tenants={tenants}
        onEdit={update}
        onDelete={remove}
        onCreate={create}
      />
    </div>
  );
}
```

### Brand Theme Integration

```typescript
// Dynamic theming based on brand configuration
function useBrandTheme() {
  const { brand } = useBrand();

  useEffect(() => {
    if (!brand?.theme) return;

    const root = document.documentElement;
    
    // Apply brand theme as CSS custom properties
    root.style.setProperty('--brand-primary', brand.theme.primaryColor);
    root.style.setProperty('--brand-secondary', brand.theme.secondaryColor);
    root.style.setProperty('--brand-text', brand.theme.textColor);
    
    if (brand.theme.logoUrl) {
      root.style.setProperty('--brand-logo-url', `url(${brand.theme.logoUrl})`);
    }

    // Apply brand-specific font if configured
    if (brand.theme.fontFamily) {
      root.style.setProperty('--brand-font', brand.theme.fontFamily);
    }
  }, [brand]);

  return {
    applyTheme: (theme: BrandTheme) => {
      const root = document.documentElement;
      Object.entries(theme).forEach(([key, value]) => {
        root.style.setProperty(`--brand-${key}`, value);
      });
    },
    resetTheme: () => {
      const root = document.documentElement;
      root.style.removeProperty('--brand-primary');
      root.style.removeProperty('--brand-secondary');
      root.style.removeProperty('--brand-text');
      root.style.removeProperty('--brand-logo-url');
      root.style.removeProperty('--brand-font');
    }
  };
}
```

## Multi-Step Form State Management

The consent form wizard maintains state across steps while being brand-aware:

```typescript
// Consent wizard state management
interface ConsentWizardState {
  // Form metadata (brand-specific)
  form: ConsentForm;
  brand: Brand;
  
  // Wizard progress
  currentStep: number;
  completedSteps: number[];
  
  // Form data
  selectedProvider: Provider | null;
  acknowledgedClauses: string[];
  personalInfo: {
    name: string;
    email: string;
    phone: string;
    dateOfBirth: string;
    address?: Address;
  };
  guardianInfo?: {
    name: string;
    relationship: string;
    signature?: string;
  };
  
  // Media captures
  photos: {
    subject?: FileUpload;
    identification?: FileUpload;
  };
  signatures: {
    subject?: string;      // Base64 signature data
    guardian?: string;
  };
  
  // Validation state
  errors: Record<string, string>;
  isValid: boolean;
}

// Consent wizard reducer
function consentWizardReducer(
  state: ConsentWizardState,
  action: ConsentWizardAction
): ConsentWizardState {
  switch (action.type) {
    case 'SELECT_PROVIDER':
      return {
        ...state,
        selectedProvider: action.provider,
        errors: { ...state.errors, provider: undefined }
      };

    case 'ACKNOWLEDGE_CLAUSE':
      return {
        ...state,
        acknowledgedClauses: [
          ...state.acknowledgedClauses,
          action.clauseId
        ]
      };

    case 'UPDATE_PERSONAL_INFO':
      return {
        ...state,
        personalInfo: {
          ...state.personalInfo,
          ...action.data
        }
      };

    case 'CAPTURE_SIGNATURE':
      return {
        ...state,
        signatures: {
          ...state.signatures,
          [action.type]: action.signatureData
        }
      };

    case 'UPLOAD_PHOTO':
      return {
        ...state,
        photos: {
          ...state.photos,
          [action.photoType]: action.file
        }
      };

    case 'NEXT_STEP':
      return {
        ...state,
        currentStep: state.currentStep + 1,
        completedSteps: [
          ...state.completedSteps,
          state.currentStep
        ]
      };

    case 'PREVIOUS_STEP':
      return {
        ...state,
        currentStep: Math.max(0, state.currentStep - 1)
      };

    case 'SET_ERRORS':
      return {
        ...state,
        errors: action.errors,
        isValid: Object.keys(action.errors).length === 0
      };

    default:
      return state;
  }
}

// Usage in consent wizard component
function ConsentWizard({ formSlug }: { formSlug: string }) {
  const { brand } = useBrand();
  const [form, setForm] = useState<ConsentForm | null>(null);
  const [state, dispatch] = useReducer(consentWizardReducer, {
    form: null,
    brand,
    currentStep: 0,
    completedSteps: [],
    selectedProvider: null,
    acknowledgedClauses: [],
    personalInfo: {
      name: '',
      email: '',
      phone: '',
      dateOfBirth: ''
    },
    photos: {},
    signatures: {},
    errors: {},
    isValid: false
  });

  // Load form configuration
  useEffect(() => {
    const loadForm = async () => {
      try {
        const response = await fetch(`/api/v1/public/forms/${formSlug}`, {
          headers: {
            'X-Brand-Context': brand.slug
          }
        });
        const formData = await response.json();
        setForm(formData);
      } catch (error) {
        console.error('Failed to load form:', error);
      }
    };

    loadForm();
  }, [formSlug, brand]);

  const steps = [
    {
      name: 'Select Provider',
      component: <ProviderStep
        providers={form?.availableProviders || []}
        selected={state.selectedProvider}
        onSelect={(provider) =>
          dispatch({ type: 'SELECT_PROVIDER', provider })
        }
      />
    },
    {
      name: 'Review Terms',
      component: <ClausesStep
        clauses={form?.requiredClauses || []}
        acknowledged={state.acknowledgedClauses}
        onAcknowledge={(clauseId) =>
          dispatch({ type: 'ACKNOWLEDGE_CLAUSE', clauseId })
        }
      />
    },
    {
      name: 'Personal Information',
      component: <PersonalInfoStep
        data={state.personalInfo}
        onChange={(data) =>
          dispatch({ type: 'UPDATE_PERSONAL_INFO', data })
        }
      />
    },
    {
      name: 'Signatures & Photos',
      component: <MediaCaptureStep
        photos={state.photos}
        signatures={state.signatures}
        onPhotoCapture={(photoType, file) =>
          dispatch({ type: 'UPLOAD_PHOTO', photoType, file })
        }
        onSignatureCapture={(type, signatureData) =>
          dispatch({ type: 'CAPTURE_SIGNATURE', type, signatureData })
        }
      />
    }
  ];

  return (
    <WizardContainer
      steps={steps}
      currentStep={state.currentStep}
      onNext={() => dispatch({ type: 'NEXT_STEP' })}
      onPrevious={() => dispatch({ type: 'PREVIOUS_STEP' })}
      brandTheme={brand.theme}
    />
  );
}
```

## Code Sharing Between Admin and Public Apps

Both the admin interface and public forms share:

1. **Brand context patterns** — Same brand resolution and theming logic
2. **API client utilities** — Shared HTTP client with brand-scoping
3. **Type definitions** — Same TypeScript interfaces for all entities
4. **Form validation** — Zod schemas for consistent validation
5. **UI components** — Shared component library with brand theming

The key differences are in access patterns (admin CRUD vs public read-only) and user flows (management vs intake), but the underlying state management patterns remain consistent across the platform.
