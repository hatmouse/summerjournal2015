# PHP学习笔记
### 1.变量
#### 变量特点
1. 弱类型语言
2. 区分大小写

#### 变量类型
字符串类型，整型，浮点型和数组
```
$var = 'string';
$var = '"str"ing';
$var = '\'str\'ing';
$var = "string";
$var = '12';
$var = TRUE;
$var = 0123; //八进制
$var = 0x123; //十六进制
$var = 1.2e3; //科学计数法
$var = 4.0E-3;
$var = 'dump,$var'; //不会替换. echo=dump,$var
$var = "dump,$var"; //变量替换. $var='string';echo=dump,string
$var =<<<STR
	test
	test
	STR;
$var = array('2012'=>'good','2015'=>'perfect')
```
### 2. 常量
```
$PIE='PIE';
define($PIE,3.14);
constant('PIE');
constant($PIE); //取常量值等价与直接使用常量名
```

### 3. 赋值
```
$var='string';
$var_a=&$var; //引用，指向同一块内存
$b = $a >= 60 ? "及格": "不及格"; //三元运算符‘
$var.='string' //$var=stringstring;
ceil(0.5)==1
```

### 4. 循环
```
$var = array('2012'=>'good','2015'=>'perfect')
foreach($var as $key => $value)
```

### 5. 数组
#### 1. 索数
```
$fruit=array('a','b','c');
print_r($fruit);
$fruit=array('1'=>'a','2'=>'b','3'=>'c');
$fruit[4]='d';
$fruit=array(3=>'a','b'); //!!! $fruit[4]=='b';
```
#### 2. 关联数组
```
$fruit=array('apple'=>'苹果','banana'=>'香蕉')
```
打印数组
```
foreach($fruit as $key=>$value)
```
### 6. 函数
```
functiton func(){
    echo 'hello world.';
}
$funcname='func';
$funcname(); //可变函数，通过变量值调用函数
if(function_exists('func')){
    echo 'yes.'; //判断函数是否存在
}
```

### 7. 类
```
class Car{
    public $name='汽车';
    publi static $hello='hello,world';
    //构造函数
    function __construct(){
        echo 'construct...';
    }
    function getName(){
        return $this->name; //!!! not '$this->$name'
    }
    //类方法
    public static function getConstant(){
        //静态方法不可使$this，可以使用self,parent
        echo self::$hello;
        return 'hello world.';
    }
}

//继承
class Trunk extends Car(){
    //重写
    function __construct(){
        echo 'chil construct...';
        //显式构造父类
        parent::__construct();
    }
}
$car = new Car();
$CarName='Car';
$car = new $CarName();
echo $car->name; //调用实例对象
echo $car->getName(); //调用实例方法
Car::getConstant(); //调用类方法
```
单例模式只允许有一个全局唯一的对象
```
class Car{
    //构造函数设置为私有，禁止调用
    private function __construct(){
        echo 'construct...';
    }
    
    private static $object=NULL;
    //仅允许通过该函数返回唯一实例 public static function getInstance(){
        if(empty(self::$object)){
            self::$object=new Car();
        }
        return self::$object;
    }
}
```

#### 属性重载
通过重载__set(),__get(),__isset(),__unset()
```
class Car {
    private $ary = array();
    
    public function __set($key, $val) {
        $this->ary[$key] = $val;
    }
    
    public function __get($key) {
        if (isset($this->ary[$key])) {
            return $this->ary[$key];
        }
        return null;
    }
    
    public function __isset($key) {
        if (isset($this->ary[$key])) {
            return true;
        }
        return false;
    }
    
    public function __unset($key) {
        unset($this->ary[$key]);
    }
}
$car = new Car();
$car->name = '汽车';  //name属性动态创建并赋值
echo $car->name;
```
#### 方法重载
方法重载通过__call(),当调用不存在的方法自动转换为调用__call($name,$args),静态方法通过__callStatic()重载
```
class Car {
    public $speed = 0;
    
    public function __call($name, $args) {
        if ($name == 'speedUp') {
            $this->speed += 10;
        }
    }
}
$car = new Car();
$car->speedUp(); //调用不存在的方法会使用重载
echo $car->speed;
```

#### 对象高级特性
对象相等
```
$car_a=new Car();
$car_b=new Car();
//对象属性相等
$car_a==$car_b
//同一个对象引用
$car_a===$car_b
```
对象克隆
```
class Car {
    public $name = 'car';
    //该方法提供克隆返回对象
    public function __clone() {
        $obj = new Car();
        $obj->name = $this->name;
    }
}
$a = new Car();
$a->name = 'new car';
$b = clone $a;
var_dump($b);
```
对象序列化
```
class Car {
    public $name = 'car';
}
$a = new Car();
$str = serialize($a); //对象序列化成字符串，进行保存
$b = unserialize($str); //反序列化为对象
var_dump($b);
```

## 字符串
* 单引号的内容总被认为普通字符串

```
trim($str) //去掉首尾空格
strlen($str) //获取字符串长度
mb_strlen($str,"UTF8") // 获取中文字符串长度
substr($str,start,len) //截取字符串，起始位置，截取长度
mb_substr($str,start,len,"utf8")
strpos($str,'hello') //查找字符串位置
str_replace($find,$replace,$str) //查找字符串，替换字符串，目标字符串
$result=sprintf("%01.2f",$str) //$str=99.9 $result=99.90
implode(",",array('hello','world')) //等价python的join，合并
explode(",","hello,world") //分割字符串为array
addslashes($str) //转义字符串，'=>/'
```

## 正则表达式
```
if(preg_match('/http/i',"http://www.linevery.com")) // /作为分隔符，i表示忽略大小写，\可以对/转义

# 这则表达式格式
$p = '/\d+\-\d+/';
$str = "我的电话是010-12345678";
preg_mathch($p,$str);

# 正则表达式结果保存
$subject = "abcdef";
$pattern = '/a(.*?)d/';
preg_match($pattern, $subject, $matches);
print_r($matches); //结果为：Array ( [0] => abcd [1] => bc )

# 正则表达式调整日期格式
$string = 'April 15, 2014';
$pattern = '/(\w+) (\d+), (\d+)/i';
$replacement = '$3, ${1} $2';
echo preg_replace($pattern, $replacement, $string); //结果为：2014, April 15
```

## Cookie
```
# 设置cookie
setcookie(key,value)
# 删除cookie,只需要设置cookie过期即可
setcookie(key,'',time()-1)
# session
session_start();
$_SESSION['key']=time();
unset($_SESSION('key'));
# session_destroy并不会立即删除全部值，只有在下次访问时才会清除

#cookie加密保存
$str = serialize($userinfo); //将用户信息序列化
echo "用户信息加密前：".$str;
$str = base64_encode(mcrypt_encrypt(MCRYPT_RIJNDAEL_256, $secureKey, $str, MCRYPT_MODE_ECB));
echo "用户信息加密后：".$str;
//将加密后的用户数据存储到cookie中
setcookie('userinfo', $str);

//当需要使用时进行解密
$str = mcrypt_decrypt(MCRYPT_RIJNDAEL_256, $secureKey, base64_decode($str), MCRYPT_MODE_ECB);
```
