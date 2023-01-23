# ed-laravel-upload-ckeditor
By David Carr article https://dcblog.dev/upload-images-in-ckeditor-5-with-laravel

CKeditor 5 out-of-the-box does not come with upload capabilities. Uploading is supported with its plugins, some are official paid plugins that require subscriptions. There are a few free options.

## Base64 upload adapter

This plugin allows uploads that will convert an uploaded image into base64 data. Very easy to use but will require you to save the complete base64 code with the post, this can get long. [Documentation](https://ckeditor.com/docs/ckeditor5/latest/features/images/image-upload/base64-upload-adapter.html)

## Simple image adapter

An image upload tool. It allows uploading images to an application running on your server using the XMLHttpRequest API with a minimal editor configuration.
[Documentation](https://ckeditor.com/docs/ckeditor5/latest/features/images/image-upload/simple-upload-adapter.html)

Use the [online builder](https://ckeditor.com/ckeditor-5/online-builder/) to add the simple image adapter then download the generated bundle unzip and place the folder inside a publicly accessible place in Laravel. such as **/public/js/ckeditor5**

Then link ckeditor.js in the pages you want to use Ckeditor

```<script src="/js/ckeditor5/build/ckeditor.js"></script>```

Using Ckeditor in a textarea.
I've made a blade component called Ckeditor so I can use:

```<x-form.ckeditor id="content" wire:model="content" name="content" />```

To render the editor. I'm using Livewire and AlpineJS.
The component looks like this:

```
@props([
    'name' => '',
    'label' => '',
    'required' => false
])

@if ($label == '')
    @php
        //remove underscores from name
        $label = str_replace('_', ' ', $name);
        //detect subsequent letters starting with a capital
        $label = preg_split('/(?=[A-Z])/', $label);
        //display capital words with a space
        $label = implode(' ', $label);
        //uppercase first letter and lower the rest of a word
        $label = ucwords(strtolower($label));
    @endphp
@endif
<div wire:ignore class="mt-5">
    @if ($label !='none')
        <label for="{{ $name }}" class="block text-sm font-medium leading-5 text-gray-700 dark:text-gray-200">{{ $label }} @if ($required != '') <span class="text-red-600">*</span>@endif</label>
    @endif
    <textarea
        x-data
        x-init="
            ClassicEditor
                .create($refs.item, {
                simpleUpload: {
                    uploadUrl: '{{ url('admin/image-upload') }}'
                }
                })
                .then(editor => {
                    editor.model.document.on('change:data', () => {
                    @this.set('{{ $name }}', editor.getData());
                    })
               })
                .catch(error => {
                    console.error(error);
                });
        "
        x-ref="item"
        {{ $attributes }}
    >
        {{ $slot }}
    </textarea>
</div>
@error($name)
    <p class="error">{{ $message }}</p>
@enderror
```

the important part is:

```
simpleUpload: {
    uploadUrl: '{{ url('admin/image-upload') }}'
}
```

This tells Ckeditor where to upload files to.

My route is defined inside an auth group so you have to be authenticated in order to use the upload route.

```
Route::middleware(['web', 'auth'])->group(function () {
    //other routes
    Route::post('admin/image-upload', [UploadController::class, 'index']);
});
```

Next create an UploadController:

```
<?php namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;
use Intervention\Image\Facades\Image;

class UploadController
{
    public function index(Request $request)
    {
        if ($request->hasFile('upload')) {

            $file = $request->file('upload');
            $name = $file->getClientOriginalName();
            $name = Str::slug($name);
            $img  = Image::make($file);
            $img->stream();
            $name = str_replace('png', '', $name).'.png';

            Storage::disk('images')->put('posts/'.$name, $img);

            return response()->json([
                'url' => "/images/posts/$name"
            ]);

        }
    }
}
```

