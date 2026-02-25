# Pattern: Media Storage & Digital Signatures

## Cloudinary Integration

All media assets (client photos, digital signatures, form title images) are stored in Cloudinary with direct client-side uploads. The API server never handles image binary data — clients upload directly to Cloudinary and store the returned references in MongoDB.

### Upload Flow

```
Client (Web/Mobile)                    Cloudinary                    MongoDB
        │                                  │                           │
        │  1. Prepare upload               │                           │
        │  (SHA-1 sign request)            │                           │
        │                                  │                           │
        │  2. POST multipart/form-data     │                           │
        │  {file, api_key, timestamp,      │                           │
        │   signature, upload_preset}      │                           │
        │─────────────────────────────────→│                           │
        │                                  │                           │
        │  3. Response:                    │                           │
        │  {secure_url, public_id,         │                           │
        │   format, bytes, ...}            │                           │
        │←─────────────────────────────────│                           │
        │                                  │                           │
        │  4. Store references in submission│                           │
        │  {ConsentPhotoId: secure_url,    │                           │
        │   ConsentPhotoIdPublicId: id}    │                           │
        │──────────────────────────────────────────────────────────────→│
        │                                  │                           │
```

### Upload Authentication (SHA-1 Signing)

```javascript
// Client-side upload signing — prevents unauthorized uploads
// without exposing the full API secret

function signUpload(paramsToSign) {
    const timestamp = Math.round(Date.now() / 1000);

    // Build the string to sign (alphabetically sorted params + secret)
    const stringToSign = Object.keys(paramsToSign)
        .sort()
        .map(key => `${key}=${paramsToSign[key]}`)
        .join('&') + API_SECRET;

    // SHA-1 hash
    const signature = sha1(stringToSign);

    return { timestamp, signature };
}

// Upload to Cloudinary
async function uploadToCloudinary(file) {
    const timestamp = Math.round(Date.now() / 1000);
    const paramsToSign = { timestamp, upload_preset: UPLOAD_PRESET };
    const signature = signUpload(paramsToSign);

    const formData = new FormData();
    formData.append('file', file);
    formData.append('api_key', API_KEY);
    formData.append('timestamp', timestamp);
    formData.append('signature', signature);
    formData.append('upload_preset', UPLOAD_PRESET);

    const response = await fetch(CLOUDINARY_UPLOAD_URL, {
        method: 'POST',
        body: formData
    });

    return response.json();
    // Returns: { secure_url, public_id, format, bytes, ... }
}
```

### Asset Deletion

```javascript
// Clean up assets when submissions are deleted
async function deleteFromCloudinary(publicId) {
    const timestamp = Math.round(Date.now() / 1000);
    const signature = signUpload({
        public_id: publicId,
        timestamp
    });

    await fetch(CLOUDINARY_DESTROY_URL, {
        method: 'POST',
        body: JSON.stringify({
            public_id: publicId,
            api_key: API_KEY,
            timestamp,
            signature
        })
    });
}
```

## Digital Signature Capture

### Web (React Admin Console)

```javascript
// react-signature-canvas — canvas-based signature capture
import SignatureCanvas from 'react-signature-canvas';

function SignaturePad({ onSave }) {
    const sigRef = useRef();

    const handleSave = () => {
        if (sigRef.current.isEmpty()) return;

        // Export as base64 PNG
        const dataUrl = sigRef.current.toDataURL('image/png');

        // Upload to Cloudinary
        const cloudinaryResult = await uploadToCloudinary(dataUrl);

        onSave({
            signatureUrl: cloudinaryResult.secure_url,
            signaturePublicId: cloudinaryResult.public_id
        });
    };

    const handleClear = () => sigRef.current.clear();

    return (
        <div>
            <SignatureCanvas
                ref={sigRef}
                penColor="black"
                canvasProps={{
                    width: 500,
                    height: 200,
                    className: 'signature-canvas'
                }}
            />
            <button onClick={handleClear}>Clear</button>
            <button onClick={handleSave}>Accept Signature</button>
        </div>
    );
}
```

### Mobile (React Native Kiosk)

```javascript
// react-native-signature-canvas — touch-optimized for tablets
import SignatureScreen from 'react-native-signature-canvas';

function MobileSignature({ onSave }) {
    const sigRef = useRef();

    const handleSignature = (signature) => {
        // signature is a base64 data URL
        const cloudinaryResult = await uploadToCloudinary(signature);
        onSave({
            signatureUrl: cloudinaryResult.secure_url,
            signaturePublicId: cloudinaryResult.public_id
        });
    };

    return (
        <SignatureScreen
            ref={sigRef}
            onOK={handleSignature}
            descriptionText="Sign above"
            clearText="Clear"
            confirmText="Accept"
            webStyle={`.m-signature-pad { height: 300px; }`}
        />
    );
}
```

## Photo Capture

### Mobile (expo-camera)

```javascript
import { Camera } from 'expo-camera';

function PhotoCapture({ onCapture }) {
    const cameraRef = useRef();
    const [hasPermission, setHasPermission] = useState(null);

    useEffect(() => {
        Camera.requestCameraPermissionsAsync()
            .then(({ status }) => setHasPermission(status === 'granted'));
    }, []);

    const takePicture = async () => {
        const photo = await cameraRef.current.takePictureAsync({
            quality: 0.7,     // Balance quality vs upload size
            base64: true       // Include base64 for direct Cloudinary upload
        });

        const cloudinaryResult = await uploadToCloudinary(photo.uri);
        onCapture({
            photoUrl: cloudinaryResult.secure_url,
            photoPublicId: cloudinaryResult.public_id
        });
    };

    return (
        <Camera ref={cameraRef} type={Camera.Constants.Type.front}>
            <TouchableOpacity onPress={takePicture}>
                <Text>Take Photo</Text>
            </TouchableOpacity>
        </Camera>
    );
}
```

## Asset Reference Storage

Both photos and signatures are stored as Cloudinary references in the submission document:

```javascript
// Submission document stores URLs + public_ids
{
    ConsentPhotoId: "https://res.cloudinary.com/dn8exiz00/image/upload/v123/abc.jpg",
    ConsentPhotoIdPublicId: "abc",

    ConsentSignature: "https://res.cloudinary.com/dn8exiz00/image/upload/v456/def.png",
    ConsentSignaturePublicId: "def",

    // Guardian signature (when applicable)
    ConsentGuardianSignature: "https://res.cloudinary.com/.../ghi.png",
    ConsentGuardianSignaturePublicId: "ghi"
}
```

The `public_id` is stored separately from the URL to enable:
1. **Asset deletion** — Cloudinary requires `public_id` for destroy operations
2. **URL regeneration** — If CDN URLs change, assets can be re-referenced by `public_id`
3. **Transformation** — Cloudinary transformations can be applied dynamically using `public_id`
