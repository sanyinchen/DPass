### Question
There is a special case , for example: 

1. There are three lib-module : testa, testb and testc
2. testa compile succeed , testb compile failed , testc compile succeed

    + ./gradlew   testa:aDebug BUILD SUCCESSFUL
    + ./gradlew   testb:aDebug BUILD FAILED
    + ./gradlew   testc:aDebug BUILD SUCCESSFUL
    + ./gradlew   app:aDebug   BUILD FAILED



3. Your special case is need to testa -> testb ;  testb -> testc

##### how todo
1. Using substitute, it's easy :

	```
	substitute project(':testa') with project(':testb')
	
	substitute project(':testb') with project(':testc')
	```
2. What we hope is testa will be final replaced by testc. However, the result will not reach .

##### how to solve
You can change the redirect map , add testa -> testc. You can find detail code in 'xbuild/deps-fix.gradle'.
