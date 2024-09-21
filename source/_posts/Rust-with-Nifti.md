---
title: Rust with Nifti
date: 2024-09-21
tags: 
- Rust 
- Nifti
- C
categories:
- [gallery]
featured_image:  https://media.nga.gov/iiif/98dd3136-8113-4a69-a2b2-c7e5e855430a/full/full/0/default.jpg?attachment_filename=the_land-crab_%28cancer_ruricola%29_1991.163.46.jpg
---

# Working with NIFTI C library in Rust

Rust offers excellent interoperability with C, and in this tutorial, I will introduce the file structure I use to compile the NIfTI C library with Rust. While there are some Rust crates that natively implement the NIfTI format, the NIfTI C library remains the gold standard for NIfTI format I/O. It has been extensively tested and is widely used in a variety of fMRI visualization and analysis software. 

## Prerequisites

- **Cargo**: Ensure Rust's package manager is installed.
- **C Compiler**: Any distribution of a C compiler is needed.
- **NIfTI C Library Code**: The necessary code from the [NIfTI C library](https://github.com/NIFTI-Imaging/nifti_clib), which includes the following files:
  - `nifti1.h`
  - `nifti2_io_version.h`
  - `nifti2_io.c`
  - `nifti2_io.h`
  - `nifti2.h`
  - `znzlib_version.h`
  - `znzlib.c`
  - `znzlib.h`

## File organizations 

This is simply my preferred way of organizing the project, but you can structure it according to your own needs. Just make sure the paths are correctly set in the following sections, and everything should work as expected.

```
.
├── Cargo.lock
├── Cargo.toml
├── README.md
├── build.rs
├── c_external
│   ├── nifti1.h
│   ├── nifti2.h
│   ├── nifti2_io.c
│   ├── nifti2_io.h
│   ├── nifti2_io_version.h
│   ├── znzlib.c
│   ├── znzlib.h
│   └── znzlib_version.h
├── src
    ├── main.rs
    └── nifti
        ├── mod.rs
        ├── nifti.rs
        └── nifti_io.rs
```



## Tutorial

First, we need to create a rust project. We can do this by running the following command:

```bash
cargo new nifti_rust
```

Next, we need to move the NIfTI C library files to the `c_external` directory. Please download the NIfTI C library from the [official repository](https://github.com/NIFTI-Imaging/nifti_clib) and copy the files listed in the prerequisites section to the `c_external` directory.

We need a build script to compile the C code. Create a `build.rs` file in the root directory and add the following code:

```rust
fn main() {
    cc::Build::new()
        .file("c_external/nifti2_io.c")
        .file("c_external/znzlib.c")
        .include("c_external")
        .flag("-Wno-unused-parameter")
        .flag("-Wno-unused-variable")
        .compile("nifti_c_lib");
}
```

The two flags are used to suppress warnings that are not relevant to the our rust project. We only need the nifti2_io.c and znzlib.c files to compile the NIfTI C library. The `include` method is used to specify the directory where the header files are located. The `compile` method is used to specify the name of the compiled library.

We also need to add the `cc` build dependency to the `Cargo.toml` file. Add the following line to the `Cargo.toml` file:

```toml
[build-dependencies]
cc = "1.0"
```

Next, we need to create a module for the NIfTI C library. Create a `nifti` directory in the `src` directory. Inside the `nifti` directory, create a `mod.rs` file with the following code:

```rust
pub mod nifti;
pub mod nifti_io;
```

This will allow the `nifti` and `nifti_io` modules to be accessed from the `main.rs` file.
Because rust can not directly access C structs, we need to create rust structs that mirror the C structs. Create a `nifti.rs` file in the `nifti` directory. You can use AI to convert the C structs to Rust structs. A typical Rust struct with the same fields as the C struct would look like this:

```rust
#[repr(C)]
pub struct NiftiImage {
    pub ndim: i64,              // int64_t ndim;
    pub nx: i64,                // int64_t nx;
    pub ny: i64,                // int64_t ny;
    pub nz: i64,                // int64_t nz;
    pub nt: i64,                // int64_t nt;
}
```

Some structs may never be used directly in Rust, so you can skip them. The '#allow(dead_code)' attribute can be used to suppress warnings about unused fields in the struct.

Now we can create a `nifti_io.rs` file in the `nifti` directory. This file will implement the IO functions in Rust with the help of the C library. The functions in the `nifti_io.rs` file will be used to read and write NIfTI files. 

First we need the external crate to link the C library. Add the following line to the `nifti_io.rs` file:
    

```rust
extern crate libc;
```

In order to use external crate in Rust, we need to add the crate to the `Cargo.toml` file. Add the following line to the `Cargo.toml` file:

```toml
[dependencies]
libc = "0.2"
```

For converting the String to a C string, we need to use the `CString` type from the `std::ffi` module. Add the following line to the `nifti_io.rs` file:

```rust
use std::ffi::CString;
```

And then we need to bring the struct we created in the `nifti.rs` file into the `nifti_io.rs` file. Add the following line to the `nifti_io.rs` file:

```rust
use super::nifti::NiftiImage;
```

To access the C functions from the C library, we need to declare them as `extern` functions, for now just the read function. Add the following lines to the `nifti_io.rs` file:

```rust
extern {
    pub fn nifti_image_read(filename: *const libc::c_char) -> *mut NiftiImage;
}
```

Next we can implement the read function in Rust. Add the following code to the `nifti_io.rs` file:

```rust
pub fn read_nifti_image(hname: &str, read_data: i32) -> Option<Box<NiftiImage>> {
    let c_hname = CString::new(hname).expect("CString::new failed");
    unsafe {
        let image_ptr = nifti_image_read(c_hname.as_ptr(), read_data);
        if image_ptr.is_null() {
            None
        } else {
            Some(Box::from_raw(image_ptr))
        }
    }
}
```

People use rust for its safty, however, when using C library, sometimes we can not avoid using unsafe code. The `nifti_image_read` function is declared as `unsafe` because it calls the `nifti_image_read` C function. We use the Box::from_raw function to convert the raw pointer to a Box, which will be handled by Rust's memory management from now on. We kind of have to trust the C library to manage the memory correctly in the background. 

Now we can test the read function in the `main.rs` file. Add the following code to the `main.rs` file:

```rust
mod nifti;
fn main() {
    let image = nifti::nifti_io::read_nifti_image("test-cases/test.nii", 1);
    if let Some(image) = image {
        println!("Successfully read NIFTI image");
        println!("The dimensions are {}, {}, {}", image.nx, image.ny, image.nz);
        println!("There are a total of {} voxels", image.nvox)
    } else {
        println!("Failed to read NIFTI image.");
    }
}
```

Now we can run the program by running the following command:

```bash
cargo run
```

If everything is set up correctly, you should see the output of the program, which will display the dimensions of the NIfTI image. If you encounter any errors, make sure the paths are correctly set in the `build.rs` file and the `nifti_io.rs` file. Also, make sure the C library files are in the correct directory. 

## Summary

This tutorail quickly goes through the process of using the NIfTI C library in Rust. We created a Rust project, compiled the C library, created Rust structs to mirror the C structs, and implemented the read function in Rust. We then tested the read function in the `main.rs` file. There are more thing to consider when working with C library in Rust, such as better error handling, generic type for data field in the NIfTI image struct, and more functions to implement. But this tutorial should give you a good starting point for working with the NIfTI C library in Rust. 

