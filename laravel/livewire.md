---
description: ทำความรู้จักกับ Livewire กัน
---

# Livewire

Laravel Livewire เป็นเฟรมเวิร์กที่ช่วยให้สามารถพัฒนาเว็บแอปพลิเคชันที่มีการตอบสนองแบบเรียลไทม์ได้โดยไม่ต้องพึ่งพา JavaScript มากมาย

Livewire ทำหน้าที่เชื่อมต่อระหว่างส่วนของ PHP กับส่วนของ JavaScript ให้ทำงานร่วมกันอย่างมีประสิทธิภาพ ช่วยให้การพัฒนาให้สะดวกสบายและยืดหยุ่นมากขึ้น

ซึ่งนักพัฒนาสามารถใช้คำสั่งที่คุ้นเคยใน Laravel และสามารถปรับปรุง UX/UI ของหน้าเว็บได้อย่างรวดเร็ว ทำให้การพัฒนาเว็บแอปพลิเคชันกับ Laravel มีประสิทธิภาพมากยิ่งขึ้น

## การติดตั้ง

ก่อนเริ่มใช้งาน Livewire คุณจำเป็นต้องติดตั้งแพ็คเกจผ่าน Composer:

```bash
composer require livewire/livewire
```

จากนั้นเพิ่ม Livewire scripts และ styles ในเทมเพลต Blade หลักของคุณ:

```html
<html>
<head>
    @livewireStyles
</head>
<body>
    <!-- เนื้อหาเว็บไซต์ -->

    @livewireScripts
</body>
</html>
```

## การสร้าง Component พื้นฐาน

Livewire components ประกอบด้วยไฟล์ PHP class และ Blade template ที่ทำงานร่วมกัน สร้าง component ใหม่ด้วยคำสั่ง:

```bash
php artisan make:livewire ShowCounter
```

คำสั่งนี้จะสร้างไฟล์ 2 ไฟล์:
- `app/Http/Livewire/ShowCounter.php` - Class component
- `resources/views/livewire/show-counter.blade.php` - Template view

### ตัวอย่าง Component Counter

```php
namespace App\Http\Livewire;

use Livewire\Component;

class ShowCounter extends Component
{
    public $count = 0;
    
    public function increment()
    {
        $this->count++;
    }
    
    public function decrement()
    {
        $this->count--;
    }

    public function render()
    {
        return view('livewire.show-counter');
    }
}
```

Template สำหรับ counter component:

```html
<div>
    <div class="text-center">
        <h2 class="text-xl font-bold">Counter: {{ $count }}</h2>
        
        <div class="mt-4">
            <button wire:click="increment" 
                    class="px-4 py-2 bg-blue-500 text-white rounded">
                +
            </button>
            
            <button wire:click="decrement" 
                    class="px-4 py-2 bg-red-500 text-white rounded">
                -
            </button>
        </div>
    </div>
</div>
```

## การใช้งาน Properties และ Methods

Livewire properties สามารถเข้าถึงได้โดยตรงจาก template และอัพเดทแบบ real-time:

```php
class ShowPosts extends Component
{
    public $search = '';
    public $sortField = 'title';
    
    public function updatedSearch()
    {
        // ทำงานอัตโนมัติเมื่อ $search มีการเปลี่ยนแปลง
    }
    
    public function render()
    {
        return view('livewire.show-posts', [
            'posts' => Post::search($this->search)
                          ->orderBy($this->sortField)
                          ->paginate(10)
        ]);
    }
}
```

Template ที่เกี่ยวข้อง:

```html
<div>
    <input type="text" wire:model="search" placeholder="ค้นหาโพสต์...">
    
    <select wire:model="sortField">
        <option value="title">เรียงตามชื่อ</option>
        <option value="created_at">เรียงตามวันที่</option>
    </select>
    
    <div class="mt-4">
        @foreach($posts as $post)
            <div class="mb-4">
                <h3>{{ $post->title }}</h3>
                <p>{{ $post->excerpt }}</p>
            </div>
        @endforeach
    </div>
</div>
```

## Events และ Actions

Livewire รองรับ events หลายรูปแบบสำหรับการโต้ตอบกับผู้ใช้:

```html
<!-- การจัดการ Events พื้นฐาน -->
<button wire:click="save">บันทึก</button>
<input wire:keydown.enter="search">

<!-- การส่งพารามิเตอร์ -->
<button wire:click="delete({{ $post->id }})">ลบ</button>

<!-- การยืนยันก่อนดำเนินการ -->
<button wire:click="delete" wire:confirm="คุณแน่ใจหรือไม่?">ลบ</button>
```

## Loading States

จัดการสถานะ loading เพื่อปรับปรุง UX:

```html
<div>
    <button wire:click="savePost">
        <span wire:loading.remove>บันทึก</span>
        <span wire:loading>กำลังบันทึก...</span>
    </button>
    
    <!-- แสดง loading spinner เฉพาะการทำงานบางอย่าง -->
    <div wire:loading wire:target="savePost">
        <svg class="animate-spin h-5 w-5"><!-- SVG spinner --></svg>
    </div>
</div>
```

การใช้งาน Livewire ช่วยให้คุณสร้างส่วนติดต่อผู้ใช้แบบ dynamic โดยใช้ PHP เป็นหลัก ลดความซับซ้อนของ JavaScript และเพิ่มประสิทธิภาพในการพัฒนา
