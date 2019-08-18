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
You can change the redirect map , add testa -> testc. You can find detail code in ['xbuild/deps-fix.gradle'](https://github.com/sanyinchen/DPass/blob/master/xbuild/deps-fix.gradle).

```
import java.lang.reflect.Field

afterEvaluate {
    project.rootProject.allprojects {
        def itProject = it
        itProject.configurations.all {
            def itConfig = it
            itConfig.resolutionStrategy {

                dependencySubstitution {
                    final Map<String, ComponentSelector> selectorMap = new HashMap<>();
                    for (Action<? super DependencySubstitution> item : substitutionRules) {
                        String className = item.getClass().getName();
                        Class refectClass = item.getClass()
                        if (className.contains("ModuleMatchDependencySubstitutionAction")) {
                            Field substituteField = refectClass.getDeclaredField("substitute")
                            substituteField.setAccessible(true);
                            ComponentSelector substitute = substituteField.get(item)

                            Field moduleIdField = refectClass.getDeclaredField("moduleId")
                            moduleIdField.setAccessible(true);
                            ModuleIdentifier moduleId = moduleIdField.get(item)

                            selectorMap.put(moduleId.toString(), substitute);
                            continue
                        }
                        if (className.contains("ExactMatchDependencySubstitutionAction")) {
                            Field substituteField = refectClass.getDeclaredField("substitute")
                            substituteField.setAccessible(true);
                            ComponentSelector substitute = substituteField.get(item)

                            Field substitutedField = refectClass.getDeclaredField("substituted")
                            substitutedField.setAccessible(true);
                            ComponentSelector substituted = substitutedField.get(item)

                            selectorMap.put(substituted.getDisplayName(), substitute);

                            continue
                        }
                    }

                    it.all { dependencySubstitution ->
                        ComponentSelector request = dependencySubstitution.getRequested();
                        ComponentSelector target = dependencySubstitution.getTarget();
                        String requestDisplayName = request.getDisplayName();
                        String targetDisplayName = target.getDisplayName();
                        if (requestDisplayName == null || requestDisplayName.equals(targetDisplayName)) {
                            return
                        }

                        for (String item : selectorMap.keySet()) {
                            if (targetDisplayName.equals(item)) {
                                println("old substitute : " + request + " => " + target);
                                println("new substitute" + request + " => " + selectorMap.get(item)+"\n");
                                dependencySubstitution.useTarget(selectorMap.get(item))
                                break;
                            }
                        }

                    }

                }
            }
        }
    }
}

```
