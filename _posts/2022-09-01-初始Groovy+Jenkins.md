
## Groovy Start 


基于Java的面向对象语言

2007年1月发布了1.0版本。2.4作为当前主要版本。

Groovy对于Java开发人员来说很简单，因为Java和Groovy的语法非常相似

接下来来看看一些基本的语法：

```groovy
class Test {
    static void main(String[] args) {
        println("I am Main");
//        testString("content here~")
//        testArr()
//        testForIn()
//        testMap()
//        testDefault("test")
//        testFile()
//        testFile2()
//        testFile3()
//        testFile4()
//        testFile5()
//        testArray()
//        testArray2()
//        testRegex()
        testEntity()
    }

    static def testString(String content) {
        def test = "hello world!"
        println(test)
        println(content)
    }

    static def testArr(){
        def range=5..10
        println(range.get(4))
//        println(range.get(6))
    }

    static def testForIn(){
        int[] range = [0,3,6,1,7,29,9823,982134]
        for(data in range){
           println(data)
        }
    }

    static def testMap(){
        def employee = ['test1':1,'test2':'666','test3':4]
        for(data in employee){
            println(data)
        }
        employee.put("new1",1)
        println(employee.size())
    }
    static def testDefault(var1,var2=2,var3="3"){
        println(var1)
        println(var2)
        println(var3)
    }

    static def testFile(){
        new File("/Users/chenzy/Downloads/untitled.groovy").eachLine {s -> println "line: $s"}
    }
    static def testFile2(){
        File file = new File("/Users/chenzy/Downloads/untitled.groovy")
        println file.text
    }
    static def testFile3(){
       new File("/Users/chenzy/Downloads/untitled.groovy").withWriter("utf-8"){writer->
            writer.writeLine("test insert")
        }
        File file = new File("/Users/chenzy/Downloads/untitled.groovy")
        println file.text
    }

    static def testFile4(){
        File file = new File("/Users/chenzy/Downloads/untitled.groovy")
        println "file path : ${file.absolutePath}, file size : ${file.length()}"
    }

    static def testFile5(){
        def target = new File("/Users/chenzy/Downloads/untitled2.groovy")
        if(!target.exists()){
            target.createNewFile()
        }
        def source = new File("/Users/chenzy/Downloads/untitled.groovy")
        target << source.text
        println target.text
    }

    static def testArray(){
        def arr1=[1,2,3,4]
        def arr2 = ["5","6","7"]
        def arr3 = arr1.plus(arr2)
        arr3.forEach({ data -> println data })
    }

    static def testArray2(){
        def arr1=[1,2,3,4,"5","6"]
        def arr2 = ["5","6","7"]
        def arr3 = arr1.minus(arr2)
        arr3.forEach({ data -> println data })
    }

    static def testRegex(){
        def regex = ~"Gro{2}vy"
        println regex.matcher("Grovy").matches()
    }

    static def testEntity(){
        def user = new User()
        user.setUserId(1)
        user.setUserName("testUser")
        println user.toString()
    }
}

```

## Groovy with Jenkins

[Refer Links](https://blog.csdn.net/DynastyRumble/article/details/119208326)
