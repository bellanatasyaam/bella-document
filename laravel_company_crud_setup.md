# Laravel CRUD: Company (Bootstrap, jQuery, Select2, SweetAlert2)

Panduan ini menjelaskan langkah-langkah membuat project Laravel baru (fresh) dan CRUD untuk model **Company** dengan integrasi **Bootstrap 5**, **jQuery**, **Select2**, dan **SweetAlert2**. Semua contoh command dan file disertakan supaya bisa langsung dipakai.

---

## 1. Buat project Laravel baru

```bash
composer create-project laravel/laravel rental-system
cd rental-system
```

Buat database baru (misal `rental_system`) lalu update `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=rental_system
DB_USERNAME=root
DB_PASSWORD=
```

---

## 2. Migrasi

Jika migrasi `companies` belum ada, tambahkan migration berikut (contoh sudah ada di percakapan sebelumnya). Contoh migration `create_companies_table`:

```php
Schema::create('companies', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('address')->nullable();
    $table->string('phone')->nullable();
    $table->string('email')->nullable();
    $table->string('tax_number')->nullable();
    $table->timestamps();
});
```

Lalu jalankan:

```bash
php artisan migrate
```

---

## 3. Model `Company`

Buat model (jika belum):

```bash
php artisan make:model Company -m
```

Isi `app/Models/Company.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Company extends Model {
    protected $fillable = ['name','address','phone','email','tax_number'];
}
```

---

## 4. Layout dasar (Bootstrap, jQuery, Select2, SweetAlert2)

Buat layout `resources/views/layouts/app.blade.php`:

```blade
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>@yield('title','Company CRUD')</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/select2@4.1.0/dist/css/select2.min.css" rel="stylesheet" />
    @stack('styles')
</head>
<body>
    <div class="container py-4">
        @yield('content')
    </div>

    <script src="https://code.jquery.com/jquery-3.7.1.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/select2@4.1.0/dist/js/select2.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>

    @stack('scripts')
</body>
</html>
```

---

## 5. Controller Resource

Buat controller resource:

```bash
php artisan make:controller CompanyController --resource
```

Isi `app/Http/Controllers/CompanyController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Company;
use Illuminate\Http\Request;

class CompanyController extends Controller
{
    public function index()
    {
        $companies = Company::latest()->paginate(10);
        return view('companies.index', compact('companies'));
    }

    public function create()
    {
        return view('companies.create');
    }

    public function store(Request $request)
    {
        $request->validate([
            'name' => 'required|string|max:255'
        ]);

        Company::create($request->all());

        return redirect()->route('companies.index')
            ->with('success', 'Company created successfully.');
    }

    public function edit(Company $company)
    {
        return view('companies.edit', compact('company'));
    }

    public function update(Request $request, Company $company)
    {
        $request->validate([
            'name' => 'required|string|max:255'
        ]);

        $company->update($request->all());

        return redirect()->route('companies.index')
            ->with('success', 'Company updated successfully.');
    }

    public function destroy(Company $company)
    {
        $company->delete();
        return response()->json(['success' => true]);
    }
}
```

---

## 6. Route

Tambahkan resource route di `routes/web.php`:

```php
use App\Http\Controllers\CompanyController;

Route::resource('companies', CompanyController::class);
```

---

## 7. Views CRUD

Buat folder `resources/views/companies` dan tiga file: `index.blade.php`, `create.blade.php`, `edit.blade.php`, serta partial `form.blade.php`.

### `resources/views/companies/index.blade.php`

```blade
@extends('layouts.app')

@section('title','Company List')

@section('content')
<div class="d-flex justify-content-between mb-3">
    <h3>Company List</h3>
    <a href="{{ route('companies.create') }}" class="btn btn-primary">+ Add Company</a>
</div>

@if(session('success'))
<div class="alert alert-success">{{ session('success') }}</div>
@endif

<table class="table table-bordered table-striped">
    <thead>
        <tr>
            <th>ID</th><th>Name</th><th>Phone</th><th>Email</th><th>Action</th>
        </tr>
    </thead>
    <tbody>
        @foreach($companies as $company)
        <tr>
            <td>{{ $company->id }}</td>
            <td>{{ $company->name }}</td>
            <td>{{ $company->phone }}</td>
            <td>{{ $company->email }}</td>
            <td>
                <a href="{{ route('companies.edit',$company) }}" class="btn btn-sm btn-warning">Edit</a>
                <button class="btn btn-sm btn-danger btn-delete" data-id="{{ $company->id }}">Delete</button>
            </td>
        </tr>
        @endforeach
    </tbody>
</table>

{{ $companies->links() }}
@endsection

@push('scripts')
<script>
$(function(){
    $('.btn-delete').click(function(){
        let id = $(this).data('id');
        Swal.fire({
            title: 'Delete this company?',
            icon: 'warning',
            showCancelButton: true,
            confirmButtonText: 'Yes, delete'
        }).then((result) => {
            if (result.isConfirmed) {
                $.ajax({
                    url: '/companies/'+id,
                    type: 'DELETE',
                    data: { _token: '{{ csrf_token() }}' },
                    success: function(res){
                        if(res.success){
                            Swal.fire('Deleted!','Company removed.','success')
                                .then(()=> location.reload());
                        }
                    }
                });
            }
        });
    });
});
</script>
@endpush
```

### `resources/views/companies/create.blade.php`

```blade
@extends('layouts.app')
@section('title','Add Company')
@section('content')
<div class="card">
    <div class="card-header">Add Company</div>
    <div class="card-body">
        <form action="{{ route('companies.store') }}" method="POST">
            @csrf
            @include('companies.form')
            <button type="submit" class="btn btn-success">Save</button>
            <a href="{{ route('companies.index') }}" class="btn btn-secondary">Back</a>
        </form>
    </div>
</div>
@endsection
```

### `resources/views/companies/edit.blade.php`

```blade
@extends('layouts.app')
@section('title','Edit Company')
@section('content')
<div class="card">
    <div class="card-header">Edit Company</div>
    <div class="card-body">
        <form action="{{ route('companies.update',$company) }}" method="POST">
            @csrf @method('PUT')
            @include('companies.form',['company'=>$company])
            <button type="submit" class="btn btn-success">Update</button>
            <a href="{{ route('companies.index') }}" class="btn btn-secondary">Back</a>
        </form>
    </div>
</div>
@endsection
```

### `resources/views/companies/form.blade.php` (partial)

```blade
<div class="mb-3">
    <label>Name</label>
    <input type="text" name="name" class="form-control" value="{{ old('name',$company->name ?? '') }}" required>
</div>
<div class="mb-3">
    <label>Address</label>
    <textarea name="address" class="form-control">{{ old('address',$company->address ?? '') }}</textarea>
</div>
<div class="mb-3">
    <label>Phone</label>
    <input type="text" name="phone" class="form-control" value="{{ old('phone',$company->phone ?? '') }}">
</div>
<div class="mb-3">
    <label>Email</label>
    <input type="email" name="email" class="form-control" value="{{ old('email',$company->email ?? '') }}">
</div>
<div class="mb-3">
    <label>Tax Number</label>
    <input type="text" name="tax_number" class="form-control" value="{{ old('tax_number',$company->tax_number ?? '') }}">
</div>

@push('scripts')
<script>
$(function(){
    $('select').select2();
});
</script>
@endpush
```

---

## 8. CSRF & AJAX Delete

Contoh AJAX delete sudah menggunakan `_token` di data. Jika kamu ingin memakai header umum untuk AJAX, bisa tambahkan di layout:

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': '{{ csrf_token() }}'
    }
});
```

---

## 9. Opsional: DataTables

Jika ingin tabel yang searchable & sortable, tambahkan DataTables CDN dan inisialisasi di `index.blade.php` (opsional).

---

## 10. Penutup

Sekarang kamu punya project Laravel lengkap dengan CRUD `Company` + integrasi UI dasar. Kamu bisa kembangkan lebih lanjut: validasi lebih ketat, file upload (logo/company image), API resource, policy/permission, atau fitur import/export.

Jika mau, saya bisa:
- Tambahkan DataTables + server-side pagination.
- Buat fitur upload logo untuk Company.
- Tambah API resource dan Postman collection.


---

*File ini dibuat otomatis oleh ChatGPT pada `2025-08-20`.*

