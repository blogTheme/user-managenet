# Sistem Manajemen Pengguna dengan Role-Based Access Control (RBAC)

Dokumentasi lengkap dan panduan pengembangan untuk sistem manajemen pengguna dengan RBAC dinamis menggunakan Laravel 13 dan React.js/Vue.js.

## 📋 Daftar Isi

- [Spesifikasi & Stack Teknologi](#1-spesifikasi--stack-teknologi)
- [Fase 1: Inisiasi & Persiapan Basis Data](#2-fase-1-inisiasi--persiapan-basis-data-minggu-1)
- [Fase 2: Pengembangan Core Backend API](#3-fase-2-pengembangan-core-backend-api-minggu-2)
- [Fase 3: Integrasi & Pengembangan Frontend](#4-fase-3-integrasi--pengembangan-frontend-minggu-3)
- [Fase 4: Jaminan Kualitas](#5-fase-4-jaminan-kualitas-quality-assurance--testing)
- [Fase 5: Deployment & CI/CD](#6-fase-5-deployment--cicd-rilis)
- [Panduan Instalasi](#panduan-instalasi)

---

## 1. Spesifikasi & Stack Teknologi

Tabel ini menjadi kesepakatan awal bagi seluruh developer sebelum menulis satu baris kode pun.

| Komponen Sistem | Teknologi yang Ditetapkan | Fokus Utama |
|----------------|--------------------------|-------------|
| Backend Framework | Laravel 13 | Stabilitas, keamanan API, dan efisiensi query |
| Frontend Framework | React.js / Vue.js | Interaktivitas UI dan state management yang reaktif |
| Database | PostgreSQL / MySQL / SQLite | Integritas relasi antar tabel (User, Role, Permission) |
| Autentikasi | Laravel Sanctum | Menerbitkan dan memvalidasi Bearer Token secara ringan |
| Sistem RBAC | Spatie Laravel Permission | Menangani logika hak akses dan caching secara efisien |
| Dokumentasi API | Scribe / Swagger | Menjadi buku panduan bagi Frontend Developer |

---

## 2. Fase 1: Inisiasi & Persiapan Basis Data (Minggu 1)

Fase ini berfokus pada fondasi infrastruktur dan struktur data.

### Checklist Tugas:

- [ ] **Inisialisasi Repositori Git**
  - Buat dua repositori terpisah: `backend` (API) dan `frontend`
  - Setup branch protection rules untuk main/master branch

- [ ] **Instalasi Laravel 13**
  ```bash
  composer create-project laravel/laravel backend
  cd backend
  composer require spatie/laravel-permission
  composer require laravel/sanctum
  ```

- [ ] **Konfigurasi Package Spatie**
  ```bash
  php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
  php artisan config:clear
  ```

- [ ] **Desain Entity Relationship Diagram (ERD)**
  
  **Tabel Users:**
  - id (primary key)
  - name
  - email (unique)
  - password (hashed)
  - email_verified_at
  - remember_token
  - timestamps

  **Tabel Roles (dari Spatie):**
  - id (primary key)
  - name
  - guard_name
  - timestamps

  **Tabel Permissions (dari Spatie):**
  - id (primary key)
  - name
  - guard_name
  - timestamps

  **Tabel model_has_roles:**
  - model_type
  - model_id
  - role_id

  **Tabel model_has_permissions:**
  - model_type
  - model_id
  - permission_id

  **Tabel role_has_permissions:**
  - permission_id
  - role_id

- [ ] **Buat Database Seeder Superadmin**
  ```php
  // database/seeders/SuperAdminSeeder.php
  public function run(): void
  {
      // Create Superadmin Role
      $superAdminRole = Role::create(['name' => 'Superadmin']);
      
      // Create All Permissions
      $permissions = [
          'manage-users',
          'create-user',
          'edit-user',
          'delete-user',
          'view-user',
          'manage-roles',
          'create-role',
          'edit-role',
          'delete-role',
          'assign-permission',
          'view-reports'
      ];
      
      foreach ($permissions as $permission) {
          Permission::create(['name' => $permission]);
      }
      
      // Assign all permissions to Superadmin
      $superAdminRole->givePermissionTo($permissions);
      
      // Create Superadmin User
      $user = User::create([
          'name' => 'Super Administrator',
          'email' => 'superadmin@example.com',
          'password' => bcrypt('change_this_password_now!'),
          'email_verified_at' => now()
      ]);
      
      $user->assignRole('Superadmin');
  }
  ```

- [ ] **Buat Factory untuk Dummy Data**
  ```php
  // database/factories/UserFactory.php
  public function definition(): array
  {
      return [
          'name' => fake()->name(),
          'email' => fake()->unique()->safeEmail(),
          'password' => bcrypt('password123'),
          'email_verified_at' => now(),
      ];
  }
  ```

- [ ] **Jalankan Migration & Seeder**
  ```bash
  php artisan migrate
  php artisan db:seed --class=SuperAdminSeeder
  php artisan db:seed --class=UserFactory
  ```

---

## 3. Fase 2: Pengembangan Core Backend API (Minggu 2)

Fase ini dikerjakan murni oleh tim Backend untuk menyediakan titik akhir (endpoint) data.

### Struktur Controller:

#### AuthController
```php
// app/Http/Controllers/AuthController.php

class AuthController extends Controller
{
    // POST /api/login
    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required|string|min:8'
        ]);
        
        if (!Auth::attempt($request->only('email', 'password'))) {
            return response()->json([
                'message' => 'Invalid login credentials'
            ], 401);
        }
        
        $user = Auth::user();
        $token = $user->createToken('auth-token')->plainTextToken;
        
        return response()->json([
            'message' => 'Login successful',
            'user' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
                'roles' => $user->getRoleNames(),
                'permissions' => $user->getAllPermissions()->pluck('name')
            ],
            'token' => $token
        ]);
    }
    
    // POST /api/logout
    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();
        
        return response()->json([
            'message' => 'Successfully logged out'
        ]);
    }
    
    // GET /api/me
    public function me(Request $request)
    {
        $user = $request->user();
        
        return response()->json([
            'user' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
                'roles' => $user->getRoleNames(),
                'permissions' => $user->getAllPermissions()->pluck('name')
            ]
        ]);
    }
}
```

#### UserController
```php
// app/Http/Controllers/UserController.php

class UserController extends Controller
{
    public function __construct()
    {
        $this->middleware(['auth:sanctum']);
        $this->middleware(['permission:view-user'])->only(['index', 'show']);
        $this->middleware(['permission:create-user'])->only('store');
        $this->middleware(['permission:edit-user'])->only('update');
        $this->middleware(['permission:delete-user'])->only('destroy');
    }
    
    // GET /api/users
    public function index(Request $request)
    {
        $users = User::with('roles.permissions')
            ->paginate($request->get('per_page', 15));
        
        return UserResource::collection($users);
    }
    
    // POST /api/users
    public function store(StoreUserRequest $request)
    {
        $validated = $request->validated();
        
        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password'])
        ]);
        
        if (isset($validated['roles'])) {
            $user->syncRoles($validated['roles']);
        }
        
        return new UserResource($user->load('roles'));
    }
    
    // PUT /api/users/{user}
    public function update(UpdateUserRequest $request, User $user)
    {
        $validated = $request->validated();
        
        $user->update([
            'name' => $validated['name'] ?? $user->name,
            'email' => $validated['email'] ?? $user->email,
        ]);
        
        if (isset($validated['password'])) {
            $user->update(['password' => Hash::make($validated['password'])]);
        }
        
        if (isset($validated['roles'])) {
            $user->syncRoles($validated['roles']);
        }
        
        return new UserResource($user->load('roles'));
    }
    
    // DELETE /api/users/{user}
    public function destroy(User $user)
    {
        // Prevent deleting superadmin
        if ($user->hasRole('Superadmin')) {
            return response()->json([
                'message' => 'Cannot delete Superadmin user'
            ], 403);
        }
        
        $user->delete();
        
        return response()->json([
            'message' => 'User deleted successfully'
        ]);
    }
}
```

#### RolePermissionController
```php
// app/Http/Controllers/RolePermissionController.php

class RolePermissionController extends Controller
{
    public function __construct()
    {
        $this->middleware(['auth:sanctum', 'role:Superadmin']);
    }
    
    // GET /api/roles
    public function index()
    {
        $roles = Role::with('permissions')->get();
        
        return response()->json([
            'roles' => $roles
        ]);
    }
    
    // POST /api/roles
    public function store(StoreRoleRequest $request)
    {
        $validated = $request->validated();
        
        $role = Role::create(['name' => $validated['name']]);
        
        if (isset($validated['permissions'])) {
            $role->syncPermissions($validated['permissions']);
        }
        
        return response()->json([
            'message' => 'Role created successfully',
            'role' => $role->load('permissions')
        ]);
    }
    
    // PUT /api/roles/{role}
    public function update(UpdateRoleRequest $request, Role $role)
    {
        $validated = $request->validated();
        
        // Prevent editing Superadmin role
        if ($role->name === 'Superadmin') {
            return response()->json([
                'message' => 'Cannot modify Superadmin role'
            ], 403);
        }
        
        $role->update(['name' => $validated['name']]);
        
        if (isset($validated['permissions'])) {
            $role->syncPermissions($validated['permissions']);
        }
        
        return response()->json([
            'message' => 'Role updated successfully',
            'role' => $role->load('permissions')
        ]);
    }
    
    // DELETE /api/roles/{role}
    public function destroy(Role $role)
    {
        if ($role->name === 'Superadmin') {
            return response()->json([
                'message' => 'Cannot delete Superadmin role'
            ], 403);
        }
        
        $role->delete();
        
        return response()->json([
            'message' => 'Role deleted successfully'
        ]);
    }
    
    // GET /api/permissions
    public function permissions()
    {
        $permissions = Permission::all();
        
        return response()->json([
            'permissions' => $permissions
        ]);
    }
}
```

### Form Request Validation:

```php
// app/Http/Requests/StoreUserRequest.php
class StoreUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // Middleware handles authorization
    }
    
    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|string|min:8|confirmed',
            'roles' => 'array',
            'roles.*' => 'exists:roles,name'
        ];
    }
}
```

### API Routes:

```php
// routes/api.php

use App\Http\Controllers\AuthController;
use App\Http\Controllers\UserController;
use App\Http\Controllers\RolePermissionController;

// Public routes
Route::post('/login', [AuthController::class, 'login']);

// Protected routes
Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::get('/me', [AuthController::class, 'me']);
    
    // User management
    Route::apiResource('users', UserController::class);
    
    // Role & Permission management (Superadmin only)
    Route::middleware(['role:Superadmin'])->group(function () {
        Route::get('/roles', [RolePermissionController::class, 'index']);
        Route::post('/roles', [RolePermissionController::class, 'store']);
        Route::put('/roles/{role}', [RolePermissionController::class, 'update']);
        Route::delete('/roles/{role}', [RolePermissionController::class, 'destroy']);
        Route::get('/permissions', [RolePermissionController::class, 'permissions']);
    });
});
```

### Generate API Documentation:

```bash
composer require knuckleswtf/scribe
php artisan vendor:publish --tag=scribe-config
php artisan scribe:generate
```

---

## 4. Fase 3: Integrasi & Pengembangan Frontend (Minggu 3)

Tim Frontend mulai bekerja menggunakan dokumentasi API yang dihasilkan pada Fase 2.

### Setup Proyek Frontend (React.js Example):

```bash
npx create-react-app frontend
cd frontend
npm install axios react-router-dom @reduxjs/toolkit react-redux
```

### Konfigurasi Axios dengan Interceptor:

```javascript
// src/utils/axios.js
import axios from 'axios';

const api = axios.create({
    baseURL: process.env.REACT_APP_API_URL || 'http://localhost:8000/api',
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
    }
});

// Request interceptor - attach token
api.interceptors.request.use(
    (config) => {
        const token = localStorage.getItem('auth_token');
        if (token) {
            config.headers.Authorization = `Bearer ${token}`;
        }
        return config;
    },
    (error) => Promise.reject(error)
);

// Response interceptor - handle auth errors
api.interceptors.response.use(
    (response) => response,
    (error) => {
        if (error.response?.status === 401) {
            // Token expired or invalid
            localStorage.removeItem('auth_token');
            localStorage.removeItem('user_data');
            window.location.href = '/login';
        }
        
        if (error.response?.status === 403) {
            // Access forbidden
            window.location.href = '/forbidden';
        }
        
        return Promise.reject(error);
    }
);

export default api;
```

### Redux Store Setup:

```javascript
// src/store/authSlice.js
import { createSlice } from '@reduxjs/toolkit';

const initialState = {
    user: null,
    token: localStorage.getItem('auth_token'),
    isAuthenticated: !!localStorage.getItem('auth_token'),
    loading: false,
    error: null
};

const authSlice = createSlice({
    name: 'auth',
    initialState,
    reducers: {
        loginStart: (state) => {
            state.loading = true;
            state.error = null;
        },
        loginSuccess: (state, action) => {
            state.loading = false;
            state.isAuthenticated = true;
            state.user = action.payload.user;
            state.token = action.payload.token;
            localStorage.setItem('auth_token', action.payload.token);
            localStorage.setItem('user_data', JSON.stringify(action.payload.user));
        },
        loginFailure: (state, action) => {
            state.loading = false;
            state.error = action.payload;
        },
        logout: (state) => {
            state.user = null;
            state.token = null;
            state.isAuthenticated = false;
            localStorage.removeItem('auth_token');
            localStorage.removeItem('user_data');
        },
        hasPermission: (state, action) => {
            const permission = action.payload;
            return state.user?.permissions?.includes(permission) || false;
        }
    }
});

export const { loginStart, loginSuccess, loginFailure, logout } = authSlice.actions;
export default authSlice.reducer;
```

### Utility Function untuk Permission Check:

```javascript
// src/utils/permissions.js
export const hasPermission = (permission) => {
    const userData = localStorage.getItem('user_data');
    if (!userData) return false;
    
    const user = JSON.parse(userData);
    return user.permissions?.includes(permission) || false;
};

export const hasRole = (role) => {
    const userData = localStorage.getItem('user_data');
    if (!userData) return false;
    
    const user = JSON.parse(userData);
    return user.roles?.includes(role) || false;
};

export const can = (permission) => hasPermission(permission);
```

### Protected Route Component:

```javascript
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { hasPermission } from '../utils/permissions';

const ProtectedRoute = ({ children, permission }) => {
    const token = localStorage.getItem('auth_token');
    
    if (!token) {
        return <Navigate to="/login" replace />;
    }
    
    if (permission && !hasPermission(permission)) {
        return <Navigate to="/forbidden" replace />;
    }
    
    return children;
};

export default ProtectedRoute;
```

### Conditional Rendering Component:

```javascript
// src/components/Can.jsx
import { hasPermission } from '../utils/permissions';

const Can = ({ permission, children, fallback = null }) => {
    if (hasPermission(permission)) {
        return children;
    }
    
    return fallback;
};

export default Can;

// Usage example:
// <Can permission="delete-user">
//     <button onClick={handleDelete}>Delete User</button>
// </Can>
```

### Halaman Manajemen Pengguna:

```javascript
// src/pages/UserManagement.jsx
import { useEffect, useState } from 'react';
import api from '../utils/axios';
import Can from '../components/Can';

const UserManagement = () => {
    const [users, setUsers] = useState([]);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
        fetchUsers();
    }, []);
    
    const fetchUsers = async () => {
        try {
            const response = await api.get('/users');
            setUsers(response.data.data);
        } catch (error) {
            console.error('Error fetching users:', error);
        } finally {
            setLoading(false);
        }
    };
    
    const handleDelete = async (userId) => {
        if (!window.confirm('Are you sure you want to delete this user?')) {
            return;
        }
        
        try {
            await api.delete(`/users/${userId}`);
            fetchUsers();
        } catch (error) {
            console.error('Error deleting user:', error);
        }
    };
    
    if (loading) return <div>Loading...</div>;
    
    return (
        <div className="user-management">
            <h1>User Management</h1>
            
            <Can permission="create-user">
                <button onClick={() => {/* navigate to create */}}>
                    Add New User
                </button>
            </Can>
            
            <table>
                <thead>
                    <tr>
                        <th>Name</th>
                        <th>Email</th>
                        <th>Roles</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {users.map(user => (
                        <tr key={user.id}>
                            <td>{user.name}</td>
                            <td>{user.email}</td>
                            <td>{user.roles.map(r => r.name).join(', ')}</td>
                            <td>
                                <Can permission="edit-user">
                                    <button>Edit</button>
                                </Can>
                                
                                <Can permission="delete-user">
                                    <button onClick={() => handleDelete(user.id)}>
                                        Delete
                                    </button>
                                </Can>
                            </td>
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    );
};

export default UserManagement;
```

### Halaman Manajemen Role:

```javascript
// src/pages/RoleManagement.jsx
import { useEffect, useState } from 'react';
import api from '../utils/axios';

const RoleManagement = () => {
    const [roles, setRoles] = useState([]);
    const [permissions, setPermissions] = useState([]);
    const [selectedRole, setSelectedRole] = useState(null);
    const [selectedPermissions, setSelectedPermissions] = useState([]);
    
    useEffect(() => {
        fetchData();
    }, []);
    
    const fetchData = async () => {
        try {
            const [rolesRes, permissionsRes] = await Promise.all([
                api.get('/roles'),
                api.get('/permissions')
            ]);
            setRoles(rolesRes.data.roles);
            setPermissions(permissionsRes.data.permissions);
        } catch (error) {
            console.error('Error fetching data:', error);
        }
    };
    
    const handlePermissionToggle = (permissionName) => {
        setSelectedPermissions(prev =>
            prev.includes(permissionName)
                ? prev.filter(p => p !== permissionName)
                : [...prev, permissionName]
        );
    };
    
    const handleSaveRole = async () => {
        try {
            if (selectedRole) {
                await api.put(`/roles/${selectedRole.id}`, {
                    name: selectedRole.name,
                    permissions: selectedPermissions
                });
            } else {
                await api.post('/roles', {
                    name: selectedRole.name,
                    permissions: selectedPermissions
                });
            }
            fetchData();
        } catch (error) {
            console.error('Error saving role:', error);
        }
    };
    
    return (
        <div className="role-management">
            <h1>Role Management</h1>
            
            <div className="roles-list">
                {roles.map(role => (
                    <div key={role.id} className="role-card">
                        <h3>{role.name}</h3>
                        <p>Permissions: {role.permissions.length}</p>
                        <button onClick={() => setSelectedRole(role)}>
                            Edit Permissions
                        </button>
                    </div>
                ))}
            </div>
            
            {selectedRole && (
                <div className="permission-editor">
                    <h2>Edit Permissions for {selectedRole.name}</h2>
                    
                    <div className="permissions-grid">
                        {permissions.map(permission => (
                            <label key={permission.id}>
                                <input
                                    type="checkbox"
                                    checked={selectedPermissions.includes(permission.name)}
                                    onChange={() => handlePermissionToggle(permission.name)}
                                />
                                {permission.name}
                            </label>
                        ))}
                    </div>
                    
                    <button onClick={handleSaveRole}>Save Changes</button>
                    <button onClick={() => setSelectedRole(null)}>Cancel</button>
                </div>
            )}
        </div>
    );
};

export default RoleManagement;
```

---

## 5. Fase 4: Jaminan Kualitas (Quality Assurance & Testing)

Fase kritis untuk mencegah celah keamanan fatal di kemudian hari.

### Automated Testing (Backend):

```php
// tests/Feature/UserManagementTest.php

class UserManagementTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_regular_user_cannot_access_role_management(): void
    {
        $user = User::factory()->create();
        $user->assignRole('User');
        
        $response = $this->actingAs($user)
            ->getJson('/api/roles');
        
        $response->assertStatus(403);
    }
    
    public function test_superadmin_can_access_role_management(): void
    {
        $user = User::factory()->create();
        $user->assignRole('Superadmin');
        
        $response = $this->actingAs($user)
            ->getJson('/api/roles');
        
        $response->assertStatus(200);
    }
    
    public function test_user_without_delete_permission_cannot_delete_users(): void
    {
        $user = User::factory()->create();
        $user->assignRole('User');
        
        $userToDelete = User::factory()->create();
        
        $response = $this->actingAs($user)
            ->deleteJson("/api/users/{$userToDelete->id}");
        
        $response->assertStatus(403);
    }
    
    public function test_cannot_delete_superadmin(): void
    {
        $superadmin = User::factory()->create();
        $superadmin->assignRole('Superadmin');
        
        $anotherSuperadmin = User::factory()->create();
        $anotherSuperadmin->assignRole('Superadmin');
        
        $response = $this->actingAs($anotherSuperadmin)
            ->deleteJson("/api/users/{$superadmin->id}");
        
        $response->assertStatus(403);
    }
    
    public function test_users_list_includes_roles_and_permissions(): void
    {
        $user = User::factory()->create();
        $user->assignRole('User');
        
        $response = $this->actingAs($user)
            ->getJson('/api/users');
        
        $response->assertStatus(200)
            ->assertJsonStructure([
                'data' => [
                    '*' => [
                        'id',
                        'name',
                        'email',
                        'roles' => [
                            '*' => ['id', 'name']
                        ]
                    ]
                ]
            ]);
    }
}
```

```php
// tests/Feature/AuthTest.php

class AuthTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_user_can_login_with_valid_credentials(): void
    {
        $user = User::factory()->create([
            'password' => bcrypt('password123')
        ]);
        
        $response = $this->postJson('/api/login', [
            'email' => $user->email,
            'password' => 'password123'
        ]);
        
        $response->assertStatus(200)
            ->assertJsonStructure([
                'message',
                'user' => ['id', 'name', 'email', 'roles', 'permissions'],
                'token'
            ]);
    }
    
    public function test_user_cannot_login_with_invalid_credentials(): void
    {
        $response = $this->postJson('/api/login', [
            'email' => 'nonexistent@example.com',
            'password' => 'wrongpassword'
        ]);
        
        $response->assertStatus(401);
    }
    
    public function test_authenticated_user_can_get_profile(): void
    {
        $user = User::factory()->create();
        $user->assignRole('User');
        
        $response = $this->actingAs($user)
            ->getJson('/api/me');
        
        $response->assertStatus(200)
            ->assertJson([
                'user' => [
                    'id' => $user->id,
                    'name' => $user->name,
                    'email' => $user->email
                ]
            ]);
    }
}
```

### Running Tests:

```bash
# Run all tests
php artisan test

# Run specific test file
php artisan test tests/Feature/UserManagementTest.php

# Run with coverage
php artisan test --coverage
```

### Optimasi Query (Eager Loading):

```php
// Pastikan semua controller menggunakan eager loading
// Contoh di UserController:

public function index(Request $request)
{
    // GOOD: Menggunakan eager loading untuk menghindari N+1 query
    $users = User::with('roles.permissions')
        ->paginate($request->get('per_page', 15));
    
    return UserResource::collection($users);
}

// BAD: Tidak menggunakan eager loading (akan menyebabkan N+1 query)
// $users = User::paginate(15);
```

### Konfigurasi Cache Spatie:

```php
// config/permission.php

return [
    'cache' => [
        'expiration_time' => \DateInterval::createFromDateString('24 hours'),
        'key' => 'spatie.permission.cache',
        'model_key' => 'name',
        'store' => 'default',
    ],
];
```

```bash
# Clear permission cache setelah perubahan
php artisan permission:cache-reset
```

### User Acceptance Testing (UAT) Checklist:

- [ ] **Login/Logout**
  - [ ] User dapat login dengan kredensial valid
  - [ ] User ditolak dengan kredensial invalid
  - [ ] Token tersimpan dengan benar di localStorage
  - [ ] Logout menghapus token dan redirect ke login

- [ ] **Permission-based UI**
  - [ ] Tombol "Hapus User" tersembunyi untuk user tanpa permission `delete-user`
  - [ ] Tombol "Edit Role" hanya terlihat untuk Superadmin
  - [ ] Menu "Manajemen Role" hanya muncul untuk Superadmin

- [ ] **API Security**
  - [ ] Request tanpa token ditolak (401)
  - [ ] Request dengan token expired ditolak (401)
  - [ ] Request tanpa permission yang cukup ditolak (403)
  - [ ] Superadmin tidak dapat dihapus

- [ ] **Data Integrity**
  - [ ] Email harus unique
  - [ ] Password minimal 8 karakter
  - [ ] Role assignment berfungsi dengan benar
  - [ ] Permission sync berfungsi dengan benar

---

## 6. Fase 5: Deployment & CI/CD (Rilis)

Fase akhir untuk menaikkan sistem ke lingkungan Production.

### Server Setup (Ubuntu):

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Nginx
sudo apt install nginx -y

# Install PHP and extensions
sudo apt install php8.3 php8.3-fpm php8.3-mysql php8.3-pgsql \
    php8.3-curl php8.3-gd php8.3-mbstring php8.3-xml \
    php8.3-zip php8.3-bcmath php8.3-redis -y

# Install Composer
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer

# Install Node.js (untuk frontend build)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
```

### Nginx Configuration:

```nginx
# /etc/nginx/sites-available/rbac-system

server {
    listen 80;
    server_name your-domain.com;
    root /var/www/rbac-system/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### GitHub Actions CI/CD Pipeline:

```yaml
# .github/workflows/deploy.yml

name: Deploy to Production

on:
  push:
    branches: [main, master]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: secret
          MYSQL_DATABASE: testing
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, xml, ctype, iconv, mysql
      
      - name: Install Dependencies
        run: composer install --no-progress --prefer-dist --optimize-autoloader
      
      - name: Copy Environment File
        run: cp .env.example .env
      
      - name: Generate Application Key
        run: php artisan key:generate
      
      - name: Run Tests
        run: php artisan test
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: secret
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy via SSH
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/rbac-system
            git pull origin main
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
            php artisan permission:cache-reset
            sudo systemctl reload php8.3-fpm
            sudo systemctl reload nginx
```

### Environment Variables (.env):

```env
APP_NAME="RBAC System"
APP_ENV=production
APP_KEY=base64:...
APP_DEBUG=false
APP_URL=https://your-domain.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=rbac_production
DB_USERNAME=rbac_user
DB_PASSWORD=strong_password_here

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

SANCTUM_STATEFUL_DOMAINS=your-domain.com
```

### SSL/HTTPS Setup dengan Let's Encrypt:

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Obtain and install SSL certificate
sudo certbot --nginx -d your-domain.com -d www.your-domain.com

# Auto-renewal is configured automatically
# Test renewal with:
sudo certbot renew --dry-run
```

### Zero-Downtime Deployment Script:

```bash
#!/bin/bash
# deploy.sh

set -e

echo "🚀 Starting deployment..."

# Pull latest code
git pull origin main

# Install dependencies
composer install --no-dev --optimize-autoloader

# Run migrations
php artisan migrate --force

# Clear and cache configurations
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Reset permission cache
php artisan permission:cache-reset

# Build frontend assets (if applicable)
# cd ../frontend && npm run build

# Reload PHP-FPM gracefully
sudo systemctl reload php8.3-fpm

echo "✅ Deployment completed successfully!"
```

---

## Panduan Instalasi

### Backend Setup:

```bash
# Clone repository
git clone <backend-repo-url> backend
cd backend

# Install dependencies
composer install

# Copy environment file
cp .env.example .env

# Generate application key
php artisan key:generate

# Configure database in .env file
# DB_CONNECTION=mysql
# DB_DATABASE=rbac_system
# DB_USERNAME=root
# DB_PASSWORD=secret

# Run migrations
php artisan migrate

# Seed superadmin and permissions
php artisan db:seed --class=SuperAdminSeeder

# Start development server
php artisan serve
```

### Frontend Setup:

```bash
# Clone repository
git clone <frontend-repo-url> frontend
cd frontend

# Install dependencies
npm install

# Copy environment file
cp .env.example .env

# Configure API URL in .env
# REACT_APP_API_URL=http://localhost:8000/api

# Start development server
npm start
```

---

## 📞 Support & Contact

Untuk pertanyaan atau issue terkait pengembangan, silakan buat issue di repository atau hubungi tim development.

---

**Dibuat dengan ❤️ oleh Tim Development**

*Last Updated: {{ date }}*
