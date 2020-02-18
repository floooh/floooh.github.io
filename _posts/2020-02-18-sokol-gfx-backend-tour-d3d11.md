---
layout: post
title: "sokol_gfx.h Backend Tour: D3D11"
---

This is part 2 of the mini-series which looks under the hood of the
[sokol_gfx.h](https://github.com/floooh/sokol/blob/master/sokol_gfx.h)
3D-API backends.

[Part 1](https://floooh.github.io/2020/02/17/sokol-gfx-backend-tour-gl.html) started with a general overview of how backends are implemented in sokol-gfx and then looked at the GL backend in detail, and this post will be all about the Direct3D11 backend.

The D3D11 backend is probably the most straightforward and "boring" of the
(currently) 3 sokol-gfx backends. The D3D11 API is much less "granular" and
(quite frankly) better designed than OpenGL, and while the sokol-gfx API maps
slightly better to Metal than D3D11, D3D11 has the advantage over Metal that it
offers a simple C-API instead of Metal's "managed" Objective-C API, which might
be more convenient when used from Objective-C, but requires quite a bit of
'interface boilerplate' in mixed C/Obj-C code. But more on that in the next blog
post.

Even though the D3D11 backend is very simple and straightforward, there's one 'feature'
of the D3D11 backend which I'm not quite happy with:

D3D11 requires a concrete vertex shader bytecode blob when creating an 'input
layout' (D3D's name for the vertex layout). It kinda makes sense because D3D
validates the vertex shader inputs against the input layout description, and
can immediately provide validation errors instead of having to wait until the
shader and input layout is used together for rendering. To make this work, the
_sg_d3d11_create_shader() backend function must perform a single dynamic
allocation to keep the vertex shader blob around for
_sg_d3d11_create_pipeline() where the D3D11 input layout is created. This is
the only place outside of sg_setup() where sokol-gfx needs to dynamically
allocate memory.

Just like the GL backend, the D3D11 backend is made of 7 structs:

* _sg_d3d11_buffer_t
* _sg_d3d11_image_t
* _sg_d3d11_shader_t
* _sg_d3d11_pipeline_t
* _sg_d3d11_pass_t
* _sg_d3d11_context_t
* _sg_d3d11_backend_t

...and 28 'relevant' backend functions (out of 30, two of them being minor
helper functions). 

Here's how those D3D11 backend functions map to D3D11 API functins:

### _sg_d3d11_setup_backend()

- **ID3D11Device_CheckFormatSupport()** is called for each sokol-gfx pixel format
to gather the pixel format capabilities for ```sg_query_pixelformat()```

### _sg_d3d11_discard_backend()

...nothing here.

### _sg_d3d11_reset_state_cache()

- clear all "device state slots", the main reason for this is clearing any
refcounting-references held by D3D11.
    - **ID3D11DeviceContext_OMSetRenderTargets()**
    - **ID3D11DeviceContext_RSSetState()**
    - **ID3D11DeviceContext_OMSetDepthStencilState()**
    - **ID3D11DeviceContext_OMSetBlendState()**
    - **ID3D11DeviceContext_IASetVertexBuffers()**
    - **ID3D11DeviceContext_IASetIndexBuffer()**
    - **ID3D11DeviceContext_IASetInputLayout()**
    - **ID3D11DeviceContext_VSSetShader()**
    - **ID3D11DeviceContext_PSSetShader()**
    - **ID3D11DeviceContext_VSSetConstantBuffers()**
    - **ID3D11DeviceContext_PSSetConstantBuffers()**
    - **ID3D11DeviceContext_VSSetShaderResources()**
    - **ID3D11DeviceContext_PSSetShaderResources()**
    - **ID3D11DeviceContext_VSSetSamplers()**
    - **ID3D11DeviceContext_PSSetSamplers()**

### _sg_d3d11_create_context()

...nothing (the concept of multiple contexts isn't really implemented in the D3D11
backend, it's currently only really useful with multi-window GLFW applications)

### _sg_d3d11_destroy_context()

...nothing

### _sg_d3d11_activate_context()

- clear all device state, same as _sg_d3d11_reset_state_cache()

### _sg_d3d11_create_buffer()

- **ID3D11Buffer_AddRef()** (if an external D3D11 buffer is injected) OR
- **ID3D11Device_CreateBuffer()**

### _sg_d3d11_destroy_buffer()

- **ID3D11Buffer_Release()**

### _sg_d3d11_create_image()

- if image is a depth-stencil surface:
    - **ID3D11Device_CreateTexture()**
- otherwise:
    - if not a 3D texture:
        - **ID3D11Texture2D_AddRef()** (if native D3D11 texture injected) OR
        - **ID3D11Device_CreateTexture2D()**
    - otherwise (if 3D texture):
        - **ID3D11Texture3D_AddRef()** (if native D3D11 texture injected) OR
        - **ID3D11Device_CreateTexture3D()**
    - **ID3D11Device_CreateShaderResourceView()**
- if MSAA render target texture, create a separate MSAA texture:
    - **ID3D11Device_CreateTexture2D()**
- finally create sampler, NOTE that D3D11 automatically manages a shared-pool for samplers
  (e.g. creating two samplers with the same attributes will return the same sampler object):
    - **ID3D11Device_CreateSamplerState()**

### _sg_d3d11_destroy_image()

- **ID3D11Texture2D_Release()** OR
- **ID3D11Texture3D_Release()**
- if MSAA render target: **ID3D11Texture2D_Release()**
- **ID3D11ShaderResourceView_Release()**
- **ID3D11SamplerState_Release()**

### _sg_d3d11_create_shader()

- for each shader stage:
    - for each uniform block, create a D3D11 constant buffer object:
        - **ID3D11Device_CreateBuffer()**
    - if shader source code provided:
        - if not happened yet: dynamically load **d3dcompiler_47.dll**
        - **D3DCompile()** (HLSL => DX bytecode)
        - if failed: ID3D10Blob_Release()
- **ID3D11Device_CreateVertexShader()**
- **ID3D11Device_CreatePixelShader()**
- **SOKOL_MALLOC()** (dynamically allocate memory for the compiler vertex shader blob,
this will be needed later in **_sg_d3d11_create_pipeline()**
- if compiled from source: 2x **ID3D10Blob_Release()**

### _sg_d3d11_destroy_shader()

- **ID3D11VertexShader_Release()**
- **ID3D11PixelShader_Release()**
- **SOKOL_FREE()**: free the dynamically allocated memory for the vertex shader blob
- for each shader stage:
    - for each uniform block:
        - **ID3D11Buffer_Release()**

### _sg_d3d11_create_pipeline()

- **ID3D11Device_CreateInputLayout()**
- **ID3D11Device_CreateRasterizerState()**
- **ID3D11Device_CreateDepthStencilState()**
- **ID3D11Device_CreateBlendState()**

### _sg_d3d11_destroy_pipeline()

- **ID3D11InputLayout_Release()**
- **ID3D11RasterizerState_Release()**
- **ID3D11DepthStencilState_Release()**
- **ID3D11BlendState_Release()**

### _sg_d3d11_create_pass()

- for each color attachment:
    - **ID3D11Device_CreateRenderTargetView()**
- **ID3D11Device_CreateDepthStencilView()** (if depth-stencil-attachment exists)

### _sg_d3d11_destroy_pass()

- for each color attachment:
    - **ID3D11RenderTargetView_Release()**
- **ID3D11DepthStencilView_Release()** (if depth-stencil-attachment exists)

### _sg_d3d11_begin_pass()

- **ID3D11DeviceContext_OMSetRenderTargets()**
- **ID3D11DeviceContext_RSSetViewports()**
- **ID3D11DeviceContext_RSSetScissorRects()**
- for each color attachment with SG_ACTION_CLEAR:
    - **ID3D11DeviceContext_ClearRenderTargetView()**
- if depth-stencil-attachment exists and has SG_ACTION_CLEAR:
    - **ID3D11DeviceContext_ClearDepthStencilVIew()**

### _sg_d3d11_end_pass()

- for each MSAA color attachment:
    - **ID3D11Context_ResolveSubresource()**
- clear all "device slots" (same as _sg_d3d11_reset_state_cache())

### _sg_d3d11_apply_viewport()

- **ID3D11DeviceContext_RSSetViewports()**

### _sg_d3d11_apply_scissor_rect()

- **ID3D11DeviceContext_RSSetScissorRects()**

### _sg_d3d11_apply_pipeline()

- **ID3D11DeviceContext_RSSetState()**
- **ID3D11DeviceContext_OMSetDepthStencilState()**
- **ID3D11DeviceContext_OMSetBlendState()**
- **ID3D11DeviceContext_IASetPrimitiveTopology()**
- **ID3D11DeviceContext_IASetInputLayout()**
- **ID3D11DeviceContext_VSSetShader()**
- **ID3D11DeviceContext_VSSetConstantBuffers()**
- **ID3D11DeviceContext_PSSetShader()**
- **ID3D11DeviceContext_PSSetConstantBuffers()**

### _sg_d3d11_apply_bindings()

- **ID3D11DeviceContext_IASetVertexBuffers()**
- **ID3D11DeviceContext_IASetIndexBuffer()**
- **ID3D11DeviceContext_VSSetShaderResources()**
- **ID3D11DeviceContext_VSSetSamplers()**
- **ID3D11DeviceContext_PSSetShaderResources()**
- **ID3D11DeviceContext_PSSetSamplers()**

### _sg_d3d11_apply_uniforms()

- **ID3D11DeviceContext_UpdateSubresource()**

### _sg_d3d11_draw()

- **ID3D11DeviceContext_DrawIndexed()** OR
- **ID3D11DeviceContext_DrawIndexedInstanced()** OR
- **ID3D11DeviceContext_Draw()** OR
- **ID3D11DeviceContext_DrawInstances()**

### _sg_d3d11_commit()

...nothing.

### _sg_d3d11_update_buffer(), _sg_d3d11_append_buffer()

- **ID3D11DeviceContext_Map()**
- **ID3D11DeviceContext_Unmap()**

### _sg_d3d11_update_image()

- for each face (1 or 6)
    - for each array- or depth-slice:
        - for each mipmap:
            - **ID3D11DeviceContext_Map()**
            - **ID3D11DeviceContext_Unmap()**

...and that's it already for the D3D11 backend tour. Next up: Metal!
