# Pattern: Media Storage & Digital Signatures

## MongoDB Native File Storage

All media assets (client photos, digital signatures, form assets) are stored directly in MongoDB using GridFS, providing complete brand isolation and eliminating external service dependencies. The platform handles image uploads through the API with brand-aware storage and retrieval.

### Upload Flow

```
Client (Web/Mobile)                    API Gateway                    MongoDB GridFS
        │                                  │                           │
        │  1. Prepare upload with brand    │                           │
        │  context and CSRF token          │                           │
        │                                  │                           │
        │  2. POST multipart/form-data     │                           │
        │  {file, brandId, metadata}       │                           │
        │─────────────────────────────────→│                           │
        │                                  │                           │
        │                                  │  3. Validate brand scope │
        │                                  │  and file constraints    │
        │                                  │                           │
        │                                  │  4. Store in GridFS with │
        │                                  │  brand isolation         │
        │                                  │─────────────────────────→│
        │                                  │                           │
        │  5. Response:                    │  MongoDB ObjectId and    │
        │  {fileId, url, metadata}         │  file metadata           │
        │←─────────────────────────────────│←─────────────────────────│
        │                                  │                           │
```

### Brand-Aware File Upload

```typescript
// File upload service with brand isolation
export class FileUploadService {
  async uploadFile(
    file: File,
    brandId: string,
    category: 'signature' | 'photo' | 'document'
  ): Promise<FileUploadResult> {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('brandId', brandId);
    formData.append('category', category);
    formData.append('metadata', JSON.stringify({
      originalName: file.name,
      contentType: file.type,
      uploadedAt: new Date().toISOString()
    }));

    const response = await fetch('/api/v1/files/upload', {
      method: 'POST',
      body: formData,
      credentials: 'include', // Include CSRF cookie
      headers: {
        'X-Brand-Context': brandId
      }
    });

    if (!response.ok) {
      throw new Error(`Upload failed: ${response.statusText}`);
    }

    return response.json();
  }

  async getFileUrl(fileId: string, brandId: string): Promise<string> {
    return `/api/v1/files/${fileId}?brand=${brandId}`;
  }

  async deleteFile(fileId: string, brandId: string): Promise<void> {
    await fetch(`/api/v1/files/${fileId}`, {
      method: 'DELETE',
      credentials: 'include',
      headers: {
        'X-Brand-Context': brandId
      }
    });
  }
}
```

### Server-Side File Handling

```typescript
// Express route for brand-aware file uploads
import { GridFSBucket, MongoClient } from 'mongodb';
import multer from 'multer';

const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB limit
    files: 1
  },
  fileFilter: (req, file, cb) => {
    // Validate file types
    const allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
    if (allowedTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Invalid file type'));
    }
  }
});

router.post('/files/upload', 
  brandContext,           // Inject brand from hostname
  requireAuth,            // Require authentication
  upload.single('file'),  // Handle multipart upload
  async (req, res) => {
    try {
      const { brandId } = req;
      const { category } = req.body;
      const file = req.file;

      if (!file) {
        return res.status(400).json({ error: 'No file provided' });
      }

      // Create GridFS bucket with brand-specific naming
      const bucket = new GridFSBucket(db, {
        bucketName: `files_${brandId}`
      });

      // Upload stream with metadata
      const uploadStream = bucket.openUploadStream(file.originalname, {
        metadata: {
          brandId,
          category,
          originalName: file.originalname,
          contentType: file.mimetype,
          uploadedBy: req.user.id,
          uploadedAt: new Date()
        }
      });

      // Write file data to GridFS
      uploadStream.write(file.buffer);
      uploadStream.end();

      uploadStream.on('finish', () => {
        res.json({
          fileId: uploadStream.id.toString(),
          filename: file.originalname,
          contentType: file.mimetype,
          size: file.size,
          url: `/api/v1/files/${uploadStream.id}`
        });
      });

      uploadStream.on('error', (error) => {
        console.error('GridFS upload error:', error);
        res.status(500).json({ error: 'Upload failed' });
      });

    } catch (error) {
      console.error('File upload error:', error);
      res.status(500).json({ error: 'Upload failed' });
    }
  }
);

// File retrieval with brand isolation
router.get('/files/:fileId', 
  brandContext,
  async (req, res) => {
    try {
      const { fileId } = req.params;
      const { brandId } = req;

      const bucket = new GridFSBucket(db, {
        bucketName: `files_${brandId}`
      });

      // Find file and verify brand ownership
      const files = await bucket.find({
        _id: new ObjectId(fileId),
        'metadata.brandId': brandId
      }).toArray();

      if (files.length === 0) {
        return res.status(404).json({ error: 'File not found' });
      }

      const file = files[0];

      // Set appropriate headers
      res.set({
        'Content-Type': file.metadata.contentType,
        'Content-Length': file.length.toString(),
        'Cache-Control': 'private, max-age=3600', // 1 hour cache
      });

      // Stream file to response
      const downloadStream = bucket.openDownloadStream(file._id);
      downloadStream.pipe(res);

    } catch (error) {
      console.error('File retrieval error:', error);
      res.status(500).json({ error: 'File retrieval failed' });
    }
  }
);
```

## Digital Signature Capture

### Web-Based Signature (React)

```typescript
import { useRef, useCallback } from 'react';
import SignatureCanvas from 'react-signature-canvas';

interface SignaturePadProps {
  onSave: (signatureData: string) => void;
  brandTheme?: BrandTheme;
}

export function SignaturePad({ onSave, brandTheme }: SignaturePadProps) {
  const sigRef = useRef<SignatureCanvas>(null);

  const handleSave = useCallback(async () => {
    if (!sigRef.current || sigRef.current.isEmpty()) {
      return;
    }

    // Export as PNG data URL
    const signatureData = sigRef.current.toDataURL('image/png', 1.0);
    
    // Upload to platform storage
    const fileUploadService = new FileUploadService();
    const blob = await fetch(signatureData).then(r => r.blob());
    const file = new File([blob], 'signature.png', { type: 'image/png' });
    
    const uploadResult = await fileUploadService.uploadFile(
      file,
      brandContext.brandId,
      'signature'
    );

    onSave(uploadResult.url);
  }, [onSave, brandContext]);

  const handleClear = useCallback(() => {
    sigRef.current?.clear();
  }, []);

  return (
    <div className="signature-container" style={{
      '--signature-border': brandTheme?.primaryColor || '#ccc'
    } as React.CSSProperties}>
      <div className="signature-instructions">
        <p>Please sign in the box below:</p>
      </div>
      
      <div className="signature-canvas-container">
        <SignatureCanvas
          ref={sigRef}
          penColor={brandTheme?.textColor || '#000000'}
          backgroundColor="#ffffff"
          canvasProps={{
            width: 500,
            height: 200,
            className: 'signature-canvas'
          }}
        />
      </div>
      
      <div className="signature-controls">
        <button 
          type="button" 
          onClick={handleClear}
          className="btn-secondary"
        >
          Clear Signature
        </button>
        <button 
          type="button" 
          onClick={handleSave}
          className="btn-primary"
          style={{
            backgroundColor: brandTheme?.primaryColor
          }}
        >
          Accept Signature
        </button>
      </div>
    </div>
  );
}
```

### Mobile-Optimized Signature Capture

```typescript
// Touch-optimized signature capture for mobile devices
import { useRef, useState, useCallback } from 'react';

interface MobileSignatureProps {
  onSave: (signatureData: string) => void;
  brandTheme?: BrandTheme;
}

export function MobileSignatureCapture({ onSave, brandTheme }: MobileSignatureProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [isDrawing, setIsDrawing] = useState(false);
  const [hasSignature, setHasSignature] = useState(false);

  const startDrawing = useCallback((e: React.TouchEvent | React.MouseEvent) => {
    setIsDrawing(true);
    setHasSignature(true);
    
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    const rect = canvas.getBoundingClientRect();
    const x = 'touches' in e ? 
      e.touches[0].clientX - rect.left : 
      e.clientX - rect.left;
    const y = 'touches' in e ? 
      e.touches[0].clientY - rect.top : 
      e.clientY - rect.top;

    ctx.beginPath();
    ctx.moveTo(x, y);
  }, []);

  const draw = useCallback((e: React.TouchEvent | React.MouseEvent) => {
    if (!isDrawing) return;
    e.preventDefault();

    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    const rect = canvas.getBoundingClientRect();
    const x = 'touches' in e ? 
      e.touches[0].clientX - rect.left : 
      e.clientX - rect.left;
    const y = 'touches' in e ? 
      e.touches[0].clientY - rect.top : 
      e.clientY - rect.top;

    ctx.lineWidth = 2;
    ctx.strokeStyle = brandTheme?.textColor || '#000000';
    ctx.lineCap = 'round';
    ctx.lineJoin = 'round';
    ctx.lineTo(x, y);
    ctx.stroke();
  }, [isDrawing, brandTheme]);

  const stopDrawing = useCallback(() => {
    setIsDrawing(false);
  }, []);

  const clearSignature = useCallback(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    ctx.clearRect(0, 0, canvas.width, canvas.height);
    setHasSignature(false);
  }, []);

  const saveSignature = useCallback(async () => {
    const canvas = canvasRef.current;
    if (!canvas || !hasSignature) return;

    // Convert canvas to blob
    canvas.toBlob(async (blob) => {
      if (!blob) return;

      const file = new File([blob], 'signature.png', { type: 'image/png' });
      const fileUploadService = new FileUploadService();
      
      const uploadResult = await fileUploadService.uploadFile(
        file,
        brandContext.brandId,
        'signature'
      );

      onSave(uploadResult.url);
    }, 'image/png', 1.0);
  }, [hasSignature, onSave, brandContext]);

  return (
    <div className="mobile-signature-container">
      <h3 style={{ color: brandTheme?.primaryColor }}>
        Digital Signature Required
      </h3>
      
      <div className="signature-canvas-wrapper">
        <canvas
          ref={canvasRef}
          width={350}
          height={200}
          className="mobile-signature-canvas"
          onTouchStart={startDrawing}
          onTouchMove={draw}
          onTouchEnd={stopDrawing}
          onMouseDown={startDrawing}
          onMouseMove={draw}
          onMouseUp={stopDrawing}
          onMouseLeave={stopDrawing}
          style={{
            border: `2px solid ${brandTheme?.primaryColor || '#ccc'}`,
            touchAction: 'none' // Prevent scrolling while drawing
          }}
        />
        <div className="signature-placeholder">
          {!hasSignature && (
            <span>Sign here with your finger or mouse</span>
          )}
        </div>
      </div>

      <div className="signature-actions">
        <button
          type="button"
          onClick={clearSignature}
          className="btn btn-outline"
          disabled={!hasSignature}
        >
          Clear
        </button>
        <button
          type="button"
          onClick={saveSignature}
          className="btn btn-primary"
          disabled={!hasSignature}
          style={{
            backgroundColor: brandTheme?.primaryColor,
            borderColor: brandTheme?.primaryColor
          }}
        >
          Accept Signature
        </button>
      </div>
    </div>
  );
}
```

## Photo Capture Component

```typescript
// File-based photo upload with optional camera access
import { useCallback, useRef, useState } from 'react';

interface PhotoCaptureProps {
  onCapture: (fileUrl: string) => void;
  brandTheme?: BrandTheme;
  required?: boolean;
}

export function PhotoCapture({ onCapture, brandTheme, required }: PhotoCaptureProps) {
  const fileInputRef = useRef<HTMLInputElement>(null);
  const [isUploading, setIsUploading] = useState(false);
  const [previewUrl, setPreviewUrl] = useState<string | null>(null);

  const handleFileSelect = useCallback(async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    setIsUploading(true);

    try {
      // Create preview
      const preview = URL.createObjectURL(file);
      setPreviewUrl(preview);

      // Upload to platform storage
      const fileUploadService = new FileUploadService();
      const uploadResult = await fileUploadService.uploadFile(
        file,
        brandContext.brandId,
        'photo'
      );

      onCapture(uploadResult.url);
    } catch (error) {
      console.error('Photo upload failed:', error);
      alert('Photo upload failed. Please try again.');
    } finally {
      setIsUploading(false);
    }
  }, [onCapture, brandContext]);

  const triggerFileSelect = useCallback(() => {
    fileInputRef.current?.click();
  }, []);

  const removePhoto = useCallback(() => {
    setPreviewUrl(null);
    if (fileInputRef.current) {
      fileInputRef.current.value = '';
    }
  }, []);

  return (
    <div className="photo-capture-container">
      <h3 style={{ color: brandTheme?.primaryColor }}>
        Photo Upload {required && <span className="required">*</span>}
      </h3>

      <input
        ref={fileInputRef}
        type="file"
        accept="image/jpeg,image/png,image/webp"
        onChange={handleFileSelect}
        style={{ display: 'none' }}
        capture="environment" // Prefer rear camera on mobile
      />

      {previewUrl ? (
        <div className="photo-preview">
          <img src={previewUrl} alt="Captured photo" />
          <div className="photo-actions">
            <button
              type="button"
              onClick={removePhoto}
              className="btn btn-outline"
            >
              Remove
            </button>
            <button
              type="button"
              onClick={triggerFileSelect}
              className="btn btn-secondary"
            >
              Retake
            </button>
          </div>
        </div>
      ) : (
        <div 
          className="photo-placeholder"
          onClick={triggerFileSelect}
          style={{
            borderColor: brandTheme?.primaryColor,
            color: brandTheme?.textColor
          }}
        >
          <div className="photo-icon">📸</div>
          <p>Tap to {window.navigator.mediaDevices ? 'take photo' : 'select photo'}</p>
          {isUploading && <div className="uploading">Uploading...</div>}
        </div>
      )}
    </div>
  );
}
```

## File Reference Storage

All uploaded files are referenced in submission documents with brand isolation:

```typescript
// Submission document with file references
interface ConsentSubmission {
  _id: ObjectId;
  brandId: ObjectId;
  tenantId: ObjectId;
  formId: ObjectId;
  
  // Subject information
  subject: {
    name: string;
    email?: string;
    photos: {
      subject?: string;      // MongoDB file URL
      identification?: string;
    };
    signature?: string;      // MongoDB file URL
  };
  
  // Guardian information (if applicable)
  guardian?: {
    name: string;
    relationship: string;
    signature?: string;      // MongoDB file URL
  };
  
  // File metadata for cleanup
  fileReferences: {
    fileId: ObjectId;
    category: 'photo' | 'signature';
    uploadedAt: Date;
  }[];
  
  submittedAt: Date;
  ip?: string;
  userAgent?: string;
}
```

The MongoDB GridFS approach provides:
1. **Brand Isolation** - Files stored in brand-specific buckets
2. **No External Dependencies** - All data stays within MongoDB Atlas
3. **Automatic Cleanup** - File deletion when submissions are removed
4. **Scalable Storage** - GridFS handles large files efficiently
5. **Security** - Files are only accessible through authenticated API endpoints
6. **Backup Consistency** - Files included in standard MongoDB backup procedures
