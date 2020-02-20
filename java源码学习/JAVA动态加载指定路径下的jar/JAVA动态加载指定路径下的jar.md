# 前言
![完整的男人](8350955-9e18119ad1829975.jpg)

在开发项目的时候，一个优秀的项目通常不光是性能强悍、健壮就够的，通常我们还需要考虑代码的可维护性和可扩展性。可维护性与可扩展性：主要是方便代码交接之后，其他人能看懂，然后在你的代码基础上继续愉快的开发。如果说上一任程序员设计的框架非常完美，逻辑清晰，注释丰富，我一拿到就能直接上手添加新的功能，那开发体验肯定是非常棒的，也不会产生负能量链（名词解释：负能量链就是我接手了上一任程序员乱七八糟的代码，我非常不舒服，我就以完成任务为目的，也加了很多乱七八糟的代码上去，然后后面的人就能拿到更乱的代码，负能量就传递给下一个接锅侠了）。
# 引言
如果一个程序拥有足够的扩展性，那么它就可以在线完成功能的动态扩展，然后重点来了，它就需要动态加载类信息了，然后的然后我就给大家带来了一个动态加载指定文件夹下所有jar包中的拥有明确类信息的类（包含其自身和所有引入信息都是已知的）
# 方法
所用到的类都是jdk提供的，只需要装了jdk就可以使用
``` java
import java.io.File;
import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.*;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

/**
 * @author wangcanfeng
 * @description jar包加载器
 * @Date Created in 10:11-2019/9/10
 */
public class JarLoader {

    public JarLoader() {
        //NO_OP
    }

    /**
     * 功能描述: 扫描一个文件夹下面的所有jar，不包含子文件夹和子jar
     *
     * @param directoryPath
     * @return:java.util.Map<java.lang.String,java.lang.Class<?>>
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/9/12-15:21
     */
    public static Map<String, Class<?>> loadAllJarFromAbsolute(String directoryPath) throws
            NoSuchMethodException, InvocationTargetException, IllegalAccessException, IOException {

        File directory = new File(directoryPath);
        // 判断是否为文件夹，如果是文件,直接用单个jar解析的方法去解析
        if (!directory.isDirectory()) {
            // 添加jar扫描路径
            addUrl(directory);
            return loadJarFromAbsolute(directoryPath);
        }
        // 如果是文件夹，则需要循环加载当前文件夹下面的所有jar
        Map<String, Class<?>> clazzMap = new HashMap<>(16);
        File[] jars = directory.listFiles();
        if (jars != null && jars.length > 0) {
            List<String> jarPath = new LinkedList<>();
            for (File file : jars) {
                String fPath = file.getPath();
                // 只加载jar
                if (fPath.endsWith(".jar")) {
                    addUrl(file);
                    jarPath.add(fPath);
                }
            }
            if (jarPath.size() > 0) {
                for (String path : jarPath) {
                    clazzMap.putAll(loadJarFromAbsolute(path));
                }
            }
        }
        return clazzMap;
    }

    /**
     * 功能描述: 从绝对路径中加载jar包中的类
     * 扫描指定jar包前需要将这个jar的地址加入了系统类加载器的扫描列表中
     * 注意，这里只支持单个jar的加载，如果这个jar还引入了其他jar依赖，会加载失败
     * 所以只能用来加载对其他jar包没有依赖的简单对象类信息
     *
     * @param path jar包路径加载地址
     * @return:java.util.Map<java.lang.String,java.lang.Class<?>>
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/9/12-14:14
     */
    public static Map<String, Class<?>> loadJarFromAbsolute(String path) throws IOException {
        JarFile jar = new JarFile(path);
        Enumeration<JarEntry> entryEnumeration = jar.entries();
        Map<String, Class<?>> clazzMap = new HashMap<>(16);
        while (entryEnumeration.hasMoreElements()) {
            JarEntry entry = entryEnumeration.nextElement();
            // 先获取类的名称，符合条件之后再做处理，避免处理不符合条件的类
            String clazzName = entry.getName();
            if (clazzName.endsWith(".class")) {
                // 去掉文件名的后缀
                clazzName = clazzName.substring(0, clazzName.length() - 6);
                // 替换分隔符
                clazzName = clazzName.replace("/", ".");
                // 加载类,如果失败直接跳过
                try {
                    Class<?> clazz = Class.forName(clazzName);
                    // 将类名称作为键，类Class对象作为值存入mao
                    // 因为类名存在重复的可能，所以这里的类名是带包名的
                    clazzMap.put(clazzName, clazz);
                } catch (Throwable e) {
                    // 这里可能出现有些类是依赖不全的，直接跳过，不做处理，也没法做处理
                }
            }
        }
        return clazzMap;
    }


    /**
     * 功能描述: 添加需要扫描的jar包
     *
     * @param jarPath
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/9/12-15:21
     */
    public static void addUrl(File jarPath) throws NoSuchMethodException, InvocationTargetException,
            IllegalAccessException, MalformedURLException {

        URLClassLoader classLoader = (URLClassLoader) ClassLoader.getSystemClassLoader();
        // 反射获取类加载器中的addURL方法，并将需要加载类的jar路径
        Method method = URLClassLoader.class.getDeclaredMethod("addURL", URL.class);
        if (!method.isAccessible()) {
            method.setAccessible(true);
        }
        URL url = jarPath.toURI().toURL();
        // 把当前jar的路径加入到类加载器需要扫描的路径
        method.invoke(classLoader, url);
    }

}

```