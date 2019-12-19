## Tản mạn về mockery trong Unit test

Chào các bạn lập trình viên, đặc biệt là những lập trình viên PHP. Dù các bạn là ai, người mới bước vào làm công việc lập trình hay là người đã có kinh nghiệm làm việc nhiều năm thì đều đã nghe tới về unit test. 
Về khái niệm của unit test thì các bạn có thể dễ dàng tìm kiếm được nó khi search với từ khóa này trên google, có thể tóm tắt ngắn gọn lại như sau:

```
Unit Test là một loại kiểm thử phần mềm trong đó các đơn vị hay thành phần riêng lẻ của phần mềm được kiểm thử. Kiểm thử đơn vị được thực hiện trong quá trình phát triển ứng dụng. Mục tiêu của Kiểm thử đơn vị là cô lập một phần code và xác minh tính chính xác của đơn vị đó.
```

Khi bạn làm việc trong một công ty, hoặc một dự án khống quá quan trọng với việc viết unit test (code chạy được là xong) thì việc bạn có hiểu về unit test hay không cũng không còn quá quan trọng. Nhưng câu chuyện sẽ đi theo một chiều hướng khác nếu như bạn đang làm trong một công ty, hay một dự án mà đòi hỏi phải tuân thử nghiêm ngặt việc viết unit test, từng dòng code bạn viết ra phải được đảm bảo bằng unit test. Lúc này nếu bạn không biết gì về unit test thì lúc này bạn sẽ thực sự gặp vấn đề đó.

Việc một unit test có tốt hay không phụ thuộc vào số lượng asertion, hay % coverage của unit test trên số lượng function hay số line of code.

Tùy vào từng dự án mà việc yêu cầu coverage với % khác nhau. Một thư viện hỗ trợ tối đa chúng ta viết test đó là Mockery.

Nếu dự án của bạn được code chuẩn, code đẹp ngay từ đầu, thì tới lúc triển khai viết unit test, bạn xử dụng Mockery vào sẽ không gặp khó khăn gì nhiều. Nhưng với bài toán, dự án của bạn quá lớn, việc code không đảm bảo được theo chuẩn từ đầu, khi áp dụng Mockery để viết test, có những đoạn code, bạn phải update lại nó mới có thể viết test nhanh chóng được. 

Bài viết này các bạn có thể coi là tôi đang giới thiệu với mọi người về các tip, để giúp bạn viết unit test với Mockery một cách nhanh chóng nhất mà không cần sửa code quá nhiều.

## 1. Test hàm public nhưng bên trong nó gọi tới một hàm protected

Mới đọc mục lục chắc các bạn vẫn khó hình dung điều mình đang muôn nói, vậy hãy theo dõi ví dụ cụ thể sau nhé

```
<?php

class TestCase1 {
  public function index()
  {
    $this->foo();
    return true;
  }
  
  protected function foo()
  {
    return $a = 5;
  }
}
```
Bây giờ, yêu cầu bạn viết unit test cho class TestCase1 trên. Bạn có thể thấy hàm index() có gọi đến một hàm protectecd là foo(). Để test được hàm index() mà không chạy qua logic của hàm foo() chúng ta làm như sau:

```
  public function test_index_function()
  {
    $mock = Mockery::mock(TestCase1::class);
    $mock->shouldReceive('foo')->once()->andReturn(10);
    $result = $mock->index();
    // assert equals result with expect data
  }
```

Như vậy là chúng ta đã test được hàm index() mà không cần chạy qua logic của hàm foo()

## 2. test protected function

Tiếp tục với ví dụ bên trên, câu hỏi đặt ra cho bạn là làm cách nào để test được hàm foo(), chắc hẳn câu trả lời đầu tiên các bạn nghĩ tới là dùng thế này

```
public function test_foo_function()
{
  $c = new TestCase1();
  $result = $c->foo();
  // assert equals result with expect data
}

```
Nhưng các bạn chú ý một điều răng, hàm foo() được khai báo là protected, vì vậy khi bạn gọi như trên sẽ bị lỗi. Cách làm đúng như sau:

Trước hết, hãy tạo một funciton mới trong class test của bạn

```
/**
     * Get method for testing protected/private methods.
     *
     * @param string $methodName
     *
     * @return \ReflectionMethod
     *
     * @throws \ReflectionException
     */
    protected static function getMethod($methodName)
    {
        $class = new ReflectionClass(SendTemplateService::class);
        $method = $class->getMethod($methodName);
        $method->setAccessible(true);
        return $method;
    }
```

Tiếp theo, để test được hàm protected 

```
public function test_foo_function()
{
  $m = Mockery::mock(TestCase1::class);
  $result = self::getMethod('foo')
    ->invoke(
        $m
    );
  // assert equals result with expect data
  
```

Trong trường hợp hàm foo() có tham số truyền vào, thì bạn hãy truyền thêm tham số vào hàm test ngay sau biết $m, nhé.

## 3. Test class với biến protected

Khi code của bạn đang được viết thế này

```

<? php

class TestService
{ 
  public function getData($user)
  {
    // logic of this function
  }
}

<? php

use TestService;

class TestController
{
  protected $user;
  
  protected $service;
  
  public function __construct(TestService $service){
    $this->service = $service;
  }
  
  public function index()
  {
    $this->service->getData($this->user);
    // logic of this function
  }
}
```

Khi viết test cho hàm index, theo cách thông thường:

```
  $c = new TestController();
  $c->index()
```

thì bạn sẽ gặp phải lỗi báo user đang null, vậy làm cách nào để bạn có thể gán giá trị cho biến protected user 

```
<? php

use ReflectionClass;

public function test_index()
{
  $c = Mockery::mock(TestController::class);
  $userMock = 1;
  $controlerReflection = new ReflectionClass(TestController::class);
  $user = $controlerReflection->getProperty('user');
  $user->setAccessible(true);
  $user->setValue($c, $userMock);
  $result = $c->index()
  // assert equals
}

```

## 4. Test repo with subquery

```
<? php

class EloquentRepository extends Repository
{
  public function __construct(Model $model)
  {
      parent::__construct($model);
  }
  
  public function findById($id, $postId)
  {
      return $this->model->where('id', $id)
          ->whereHas('post', function ($query) use ($postId) {
              return $query->where('id', $postId);
          })
          ->count();
  }
}
```

Thông thường, để test repository chúng ta làm như sau

```
public function test_repo()
{
  $model = Mockery::mock(Model::class);
  $repo = new EloquentRepository($model);
  $model->shouldReceive('where->whereHas->count')->andReturn(1);
  $result = $repo->findById(1, 1);
  // assert equals
}
```

Với đoạn code trên, đảm bảo được hàm test của bạn chạy OK. Nhưng vấn đề là khi kiểm tra coverage thì coverage không chạy qua được dòng code 

```
->whereHas('post', function ($query) use ($postId) {
      return $query->where('id', $postId);
  })
```

Lúc này, coverage của function này sẽ không chạy được 100% line, đẫn đến coverage của method sẽ là 0%. vấn đề làn giải. xếp ép coverage > 80% mà giờ chạy được 0%. làm sao đây. Tham khảo cách dưới đây 

```
public function test_repo()
{
  $model = Mockery::mock(Model::class);
  $repo = new EloquentRepository($model);
  $model->shouldReceive('where')->andReturn($model);
  $model->shouldReceive('whereHas')->once()
  ->with('post', Mockery::on(function($whereHas) {
      // Unit Test Closure
      $mockDb = Mockery::mock('Illuminate\Database\DatabaseManager');
      $subQuery = $mockDb->shouldReceive('where')
          ->once()
          ->andReturn($mockDb);
      $whereHas->__invoke($mockDb);
      return is_callable($whereHas);
  }))
  ->andReturn($model);
  $model->shouldReceive('count')->andReturn(1);
  $result = $repo->findById(1, 1);
  // assert equals
}
```

V
