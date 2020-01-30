## Spring相关功能

---

### 全局异常处理

在Spring中配合@ExceptionHandler()和@ControllerAdvice()可以实现全局异常的集中拦截处理。

```
// assignableTypes是设置拦截特定Controller
@ControllerAdvice(assignableTypes = {ExceptionController.class})
@ExceptionHandler(value = Exception.class)// 拦截所有异常
@ResponseBody
public class GlobalExceptionHandler {

    ErrorResponse illegalArgumentResponse = new ErrorResponse(new IllegalArgumentException("参数错误!"));
    ErrorResponse resourseNotFoundResponse = new ErrorResponse(new ResourceNotFoundException("Sorry, the resourse not found!"));

    // 拦截所有异常, 这里只是为了演示，一般情况下一个方法特定处理一种异常
    @ExceptionHandler(value = Exception.class)
    public ResponseEntity<ErrorResponse> exceptionHandler(Exception e) {

        if (e instanceof IllegalArgumentException) {
            return ResponseEntity.status(400).body(illegalArgumentResponse);
        } else if (e instanceof ResourceNotFoundException) {
            return ResponseEntity.status(404).body(resourseNotFoundResponse);
        }
        return null;
    }
}
```

---

### 校验
- 常用注解
  - JSR提供的校验注解
    - @Null 被注释的元素必须为 null
    - @NotNull 被注释的元素必须不为 null
    - @AssertTrue 被注释的元素必须为 true
    - @AssertFalse 被注释的元素必须为 false
    - @Min(value) 被注释的元素必须是一个数字，其值必须大于等于指定的最小值
    - @Max(value) 被注释的元素必须是一个数字，其值必须小于等于指定的最大值
    - @DecimalMin(value) 被注释的元素必须是一个数字，其值必须大于等于指定的最小值
    - @DecimalMax(value) 被注释的元素必须是一个数字，其值必须小于等于指定的最大值
    - @Size(max=, min=) 被注释的元素的大小必须在指定的范围内
    - @Digits (integer, fraction) 被注释的元素必须是一个数字，其值必须在可接受的范围内
    - @Past 被注释的元素必须是一个过去的日期
    - @Future 被注释的元素必须是一个将来的日期
    - @Pattern(regex=,flag=) 被注释的元素必须符合指定的正则表达式

  - Hibernate Validator提供的校验注解
    - @NotBlank(message =) 验证字符串非null，且长度必须大于0
    - @Email 被注释的元素必须是电子邮箱地址
    - @Length(min=,max=) 被注释的字符串的大小必须在指定的范围内
    - @NotEmpty 被注释的字符串的必须非空
    - @Range(min=,max=,message=) 被注释的元素必须在合适的范围内

- 校验参数

  @RequestBody参数前添加@Valid注解，@PathVariable和@RequestParam需要在类上加@Validated注解，如果校验失败会抛出MethodArgumentNotValidException异常，可以在全局异常处理里面处理。
  
  ```
  @RestController
  @RequestMapping("/api")
  public class PersonController {

    @PostMapping("/person")
    public ResponseEntity<Person> getPerson(@RequestBody @Valid Person person) {
        return ResponseEntity.ok().body(person);
    }
  }

  @RestController
  @RequestMapping("/api")
  @Validated
  public class PersonController {

    @GetMapping("/person/{id}")
    public ResponseEntity<Integer> getPersonByID(@Valid @PathVariable("id") @Max(value = 5,message = "超过 id 的范围了") Integer id) {
        return ResponseEntity.ok().body(id);
    }

    @PutMapping("/person")
    public ResponseEntity<String> getPersonByName(@Valid @RequestParam("name") @Size(max = 6,message = "超过 name 的范围了") String name) {
        return ResponseEntity.ok().body(name);
    }
  }
  ```
- 手动校验

  可以通过Validator工厂类获得的Validator实例，也可以通过@Autowired直接注入的方式。但是在非Spring Component类中只能通过工厂类来获得Validator。

  ```
    @Autowired
    Validator validate

    @Test
    public void check_person_manually() {

        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        Validator validator = factory.getValidator();
        Person person = new Person();
        person.setSex("Man22");
        person.setClassId("82938390");
        person.setEmail("SnailClimb");
        Set<ConstraintViolation<Person>> violations = validator.validate(person);
        //output:
        //email 格式不正确
        //name 不能为空
        //sex 值不在可选范围
        for (ConstraintViolation<Person> constraintViolation : violations) {
            System.out.println(constraintViolation.getMessage());
        }
    }
  ```

- 自定义Validator
  
  创建一个注解
  ```
   @Target({FIELD})
   @Retention(RUNTIME)
   @Constraint(validatedBy = RegionValidator.class)
   @Documented
   public @interface Region {

       String message() default "Region 值不在可选范围内";

       Class<?>[] groups() default {};

       Class<? extends Payload>[] payload() default {};
   }
   ```

   实现ConstraintValidator接口，并重写isValid

   ```
   public class RegionValidator implements ConstraintValidator<Region, String> {

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        HashSet<Object> regions = new HashSet<>();
        regions.add("China");
        regions.add("China-Taiwan");
        regions.add("China-HongKong");
        return regions.contains(value);
    }
   }
   ```

   使用这个注解
   ```
    @Region
    private String region;
   ```