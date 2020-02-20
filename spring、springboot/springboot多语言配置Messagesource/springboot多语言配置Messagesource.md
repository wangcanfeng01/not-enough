#1.观察源码
在Springboot源码中，我们可以看到如下源码来初始化MessageSource
``` java
	protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
               //MESSAGE_SOURCE_BEAN_NAME="messageSource"
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using MessageSource [" + this.messageSource + "]");
			}
		}
		else {
			// Use empty MessageSource to be able to accept getMessage calls.
			DelegatingMessageSource dms = new DelegatingMessageSource();
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
			}
		}
	}
```

#2.处理办法
通过源码我们发现springboot在启动时，如果发现没有messageSource的bean，那么就会通过DelegatingMessageSource创建新的messageSource。但这并不是我们想要的，为了让容器加载我们自己的语言资源，那么就需要自己往容器中注入messageSource的bean
``` java
package com.wcf.funny.config.source;

import com.wcf.funny.core.utils.I18Utils;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.ReloadableResourceBundleMessageSource;

/**
 * @author wangcanfeng
 * @time 2019/3/4
 * @function 多语言资源自动载入
 **/
@Configuration
public class MessageSourceAutoConfiguration {
    /**
     * 多语言的资源文件路径,在application.properties中增加配置spring.messages.basename=classpath:i18n/message;classpath:i18n/log
     */
    @Value("${spring.messages.basename}")
    private String path;

    /**
     * 功能描述：  加载一下语言资源
     *
     * @param
     * @author wangcanfeng
     * @time 2019/3/4 22:03
     * @since v1.0
     **/
    @Bean("messageSource")
    @ConditionalOnMissingBean
    public ReloadableResourceBundleMessageSource messageSource() {
        ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
        // 用分号隔开各个语言资源路径
        String[] paths=path.split(";");
        messageSource.setBasenames(paths);
        messageSource.setDefaultEncoding("UTF-8");
        messageSource.setUseCodeAsDefaultMessage(true);
        messageSource.setFallbackToSystemLocale(false);
        I18Utils.setMessageSource(messageSource);
        return messageSource;
    }
}

```

在上述代码中，我为了后续接口调用方便，将多语言类的方法都做成了静态方法，为了可以用静态方法查询键值对应的翻译 ，我利用了这个方法来往工具类中注入语言文件I18Utils.setMessageSource(messageSource);

``` java
  private static MessageSource source;

    /**
     * 功能描述：  使用多语言方法时，必须要先注入一个多语言资源，否则无法获取多语言信息
     *
     * @param messageSource
     * @author wangcanfeng
     * @time 2019/3/4 22:22
     * @since v1.0
     **/
    public static void setMessageSource(MessageSource messageSource) {
        source = messageSource;
    }
```
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)