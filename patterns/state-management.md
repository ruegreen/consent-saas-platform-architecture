# Pattern: State Management (Redux)

## Redux Architecture

Both the React admin console and the React Native kiosk app use Redux with Redux-Thunk for async actions. The state tree mirrors the domain model — each entity type has its own slice with consistent action/reducer patterns.

### Store Structure

```javascript
// configureStore.js
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import rootReducer from '../reducers';

export default function configureStore(initialState) {
    return createStore(
        rootReducer,
        initialState,
        applyMiddleware(thunk)
    );
}
```

### State Shape

```javascript
{
    // Authentication
    userLoggedIn: [{
        _id: "...",
        Userid: "shop-owner",
        Email: "owner@shop.com",
        Company: "Ink & Iron Tattoo",
        Role: "Admin",
        SubscriptionStatus: true,
        SubscriptionType: "Monthly",
        Artists: [...],
        Clauses: [...],
        ConsentForms: [...],
        Kioskusers: [...]
    }],

    // Entity collections (fetched from API)
    tenants: [],
    tenant: {},
    artists: [],
    artist: {},
    clauses: [],
    clause: {},
    consents: [],
    consent: {},
    kioskusers: [],
    kioskuser: {},
    submissions: [],
    submission: {},
    subscriptions: [],
    subscription: {}
}
```

### Action Pattern (Redux-Thunk)

Every entity follows the same async action pattern:

```javascript
// actionTypes.js — constants prevent typo bugs
export const LOAD_ARTISTS_SUCCESS = 'LOAD_ARTISTS_SUCCESS';
export const CREATE_ARTIST_SUCCESS = 'CREATE_ARTIST_SUCCESS';
export const UPDATE_ARTIST_SUCCESS = 'UPDATE_ARTIST_SUCCESS';
export const DELETE_ARTIST_SUCCESS = 'DELETE_ARTIST_SUCCESS';

// artistActions.js — async thunks
export function loadArtists(tenantId) {
    return function (dispatch) {
        return fetch(`${API_URL}/tenants/artists/${tenantId}`, {
            method: 'GET',
            headers: new Headers({
                'Content-Type': 'application/json;charset=UTF-8',
                'x-api-key': API_KEY
            })
        })
        .then(response => response.json())
        .then(artists => dispatch(loadArtistsSuccess(artists)))
        .catch(error => console.error('Load artists failed:', error));
    };
}

function loadArtistsSuccess(artists) {
    return { type: LOAD_ARTISTS_SUCCESS, artists };
}

export function createArtist(tenantId, artist) {
    return function (dispatch) {
        return fetch(`${API_URL}/tenants/artists/${tenantId}`, {
            method: 'POST',
            headers: new Headers({
                'Content-Type': 'application/json;charset=UTF-8',
                'x-api-key': API_KEY
            }),
            body: JSON.stringify(artist)
        })
        .then(response => response.json())
        .then(result => dispatch(createArtistSuccess(result)))
        .catch(error => console.error('Create artist failed:', error));
    };
}
```

### Reducer Pattern

```javascript
// artistReducer.js
import { LOAD_ARTISTS_SUCCESS, CREATE_ARTIST_SUCCESS } from '../actions/actionTypes';

export default function artistReducer(state = [], action) {
    switch (action.type) {
        case LOAD_ARTISTS_SUCCESS:
            return action.artists;

        case CREATE_ARTIST_SUCCESS:
            return [...state, action.artist];

        case UPDATE_ARTIST_SUCCESS:
            return state.map(artist =>
                artist._id === action.artist._id ? action.artist : artist
            );

        case DELETE_ARTIST_SUCCESS:
            return state.filter(artist => artist._id !== action.artistId);

        default:
            return state;
    }
}

// index.js — combine all reducers
import { combineReducers } from 'redux';
import loginReducer from './loginReducer';
import tenantReducer from './tenantReducer';
import artistReducer from './artistReducer';
import clauseReducer from './clauseReducer';
import consentReducer from './consentReducer';
import kioskuserReducer from './kioskuserReducer';
import submissionReducer from './submissionReducer';
import subscriptionReducer from './subscriptionReducer';

export default combineReducers({
    userLoggedIn: loginReducer,
    tenants: tenantReducer,
    artists: artistReducer,
    clauses: clauseReducer,
    consents: consentReducer,
    kioskusers: kioskuserReducer,
    submissions: submissionReducer,
    subscriptions: subscriptionReducer
});
```

### API Configuration (Environment-Aware)

```javascript
// api.js — endpoint configuration
// Separate files for local vs production

// LOCAL api.js (development)
const API_BASE = 'http://localhost:3002';

// REMOTE api.js (production)
const API_BASE = 'https://consent.tattooreleaseonline.com';

export const endpoints = {
    loginUser: `${API_BASE}/api/v1/Consent/login`,
    getUserSalt: `${API_BASE}/api/v1/Consent/getUserSalt`,
    tenants: `${API_BASE}/api/v1/Consent/tenants`,
    artists: `${API_BASE}/api/v1/Consent/tenants/artists`,
    clauses: `${API_BASE}/api/v1/Consent/tenants/clauses`,
    consents: `${API_BASE}/api/v1/Consent/tenants/consents`,
    submissions: `${API_BASE}/api/v1/Consent/submissions`,
    subscriptions: `${API_BASE}/api/v1/Consent/subscriptions`,
    sendmail: `${API_BASE}/api/v1/Consent/sendmail`
};
```

## Multi-Step Wizard State

The consent form builder and client intake wizard use StepZilla (web) and react-native-wizard (mobile) for multi-step form flows. All wizard state lives in Redux to survive step navigation.

```javascript
// Wizard state preserved in Redux — not component-local state
// This enables:
// 1. Back/forward navigation without data loss
// 2. "Save and resume" for partially completed forms
// 3. Cross-step validation (Step 3 can check Step 1 selections)

// ConsentWizard component
function ConsentWizard({ consent, artists, clauses }) {
    const dispatch = useDispatch();
    const wizardState = useSelector(state => state.currentWizard);

    const steps = [
        {
            name: 'Select Artist',
            component: <ArtistStep
                artists={consent.IncludedArtists}
                selected={wizardState.selectedArtist}
                onSelect={(artist) =>
                    dispatch(updateWizardState({ selectedArtist: artist }))
                }
            />
        },
        {
            name: 'Review Clauses',
            component: <ClauseStep
                clauses={consent.IncludedClauses}
                acknowledged={wizardState.acknowledgedClauses}
                onAcknowledge={(clauseId) =>
                    dispatch(acknowledgeClauses(clauseId))
                }
            />
        },
        {
            name: 'Personal Info & Signature',
            component: <PersonalInfoStep
                formConfig={consent}
                onComplete={(submission) =>
                    dispatch(createSubmission(submission))
                }
            />
        }
    ];

    return <StepZilla steps={steps} />;
}
```

## Code Sharing Between Web and Mobile

Both the React admin console and React Native kiosk app share:

1. **Action type constants** — same `actionTypes.js` file
2. **API endpoint definitions** — same URL structure
3. **Redux patterns** — identical thunk/reducer conventions
4. **Business logic** — PBKDF2 hashing, subscription validation, age calculation

The key differences are in the view layer (React DOM vs React Native components) and platform-specific features (expo-camera vs react-webcam, react-native-signature-canvas vs react-signature-canvas).
