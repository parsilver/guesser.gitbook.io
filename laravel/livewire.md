# ทำความรู้จักกับ Livewire กัน

Laravel Livewire เป็นเฟรมเวิร์กที่ช่วยให้สามารถพัฒนาเว็บแอปพลิเคชันที่มีการตอบสนองแบบเรียลไทม์ได้โดยไม่ต้องพึ่งพา JavaScript มากมาย&#x20;

Livewire ทำหน้าที่เชื่อมต่อระหว่างส่วนของ PHP กับส่วนของ JavaScript ให้ทำงานร่วมกันอย่างมีประสิทธิภาพ ช่วยให้การพัฒนาให้สะดวกสบายและยืดหยุ่นมากขึ้น&#x20;

ซึ่งนักพัฒนาสามารถใช้คำสั่งที่คุ้นเคยใน Laravel และสามารถปรับปรุง UX/UI ของหน้าเว็บได้อย่างรวดเร็ว ทำให้การพัฒนาเว็บแอปพลิเคชันกับ Laravel มีประสิทธิภาพมากยิ่งขึ้น



ก่อนเริ่มทดลอง

โปรดตรวจสอบให้แน่ใจว่าคุณได้ติดตั้งและตั้งค่า Livewire ในโปรเจ็กต์ของคุณเรียบร้อย&#x20;

หากยังไม่ได้ติดตั้ง สามารถทำได้โดยใช้คำสั่ง:

```bash
composer require livewire/livewire

```



ยกตัวอย่างเช่น

* สร้าง Component ผ่าน Artisan command&#x20;

```
php artisan make:livewire ShowCounter
```

* ตัวอย่าง Component

```php
// In app/Http/Livewire/ShowCounter.php
namespace App\Http\Livewire;

use Livewire\Component;

class ShowCounter extends Component
{
    public $count = 0;

    public function mount()
    {
        // Do something on mount
    }
    
    public function increaseCount()
    {
        $this->count++;
    }

    public function render()
    {
        return view('livewire.show-counter');
    }
}
```

และ view ที่ `resources/views/livewire/show-counter.blade.php`&#x20;

```html
<div>
    <h1>Counter: {{ $count }}</h1>
    <button wire:click="increaseCount">Increase</button>
</div>
```



จากนั้น คุณสามารถทดสอบการทำงานของ component นี้ได้โดยเรียกใช้ใน view หลักของแอปพลิเคชัน&#x20;

เช่น ใน `resources/views/welcome.blade.php` โดยเพิ่มโค้ด

```blade
@livewire('show-counter')
```



จากนั้นเมื่อรัน Web Application, คุณจะเห็นตัวนับ (counter) ที่สามารถเพิ่มค่าได้เมื่อลองคลิกที่ปุ่ม "Increase" บนหน้าเว็บ และคุณก็จะเห็นสิ่งที่น่าอัศจรรย์เกิดขึ้น&#x20;
