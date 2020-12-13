---
title: Eclipse 配置注释模板
typora-root-url: ..
tags: Eclipse
categories: 开发工具
abbrlink: c9e92657
date: 2017-08-03 09:54:24
copyright_author: Jitwxs
---

![成果图](/images/posts/20190123223846216.png)

---

**Step1:** 打开设置：`Preferences` --> `Java` --> `CodeStyle` --> `CodeTemplates`

**Step2:** 选择 `Comments` 标签，配置注释

![](/images/posts/20190123222653126.png)

- `Files`

```ini
/** 
 * @fileName: ${file_name}  
 * @copyright_author: jitwxs
 * @date: ${date} ${time} 
 */
```

- `Types`

```ini
/** 
 * @className ${file_name}
 * @author jitwxs
 * @date ${date} ${time} 
*/ 
```

- `Methods`

```ini
/**
 * @author jitwxs
 * @date ${date} ${time} 
 * ${tags}
*/ 
```

**Step3:** 选择 `Code`

- `New Java files`

```ini
${filecomment}
${package_declaration}
/**
 * @className ${file_name}
 * @author jitwxs
 * @date ${date} ${time}   
*/
${typecomment}
${type_declaration}
```

**附录：** 下面是我导出的 xml 文件，可以直接作为配置文件导入 Eclipse：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?><templates><template autoinsert="true" context="gettercomment_context" deleted="false" description="Comment for getter method" enabled="true" id="org.eclipse.jdt.ui.text.codetemplates.gettercomment" name="gettercomment">/**
 * @return the ${bare_field_name}
 */</template><template autoinsert="true" context="delegatecomment_context" deleted="false" description="Comment for delegate methods" enabled="true" id="org.eclipse.jdt.ui.text.codetemplates.delegatecomment" name="delegatecomment">/**
 * ${tags}
 * ${see_to_target}
 */</template><template autoinsert="true" context="fieldcomment_context" deleted="false" description="Comment for fields" enabled="true" id="org.eclipse.jdt.ui.text.codetemplates.fieldcomment" name="fieldcomment">/**
 * 
 */</template><template autoinsert="true" context="constructorcomment_context" deleted="false" description="Comment for created constructors" enabled="true" id="org.eclipse.jdt.ui.text.codetemplates.constructorcomment" name="constructorcomment">/**
 * ${tags}
 */</template><template autoinsert="false" context="filecomment_context" deleted="false" description="Comment for created Java files" enabled="true" id="org.eclipse.jdt.ui.text.codetemplates.filecomment" name="filecomment">/** 
 * @fileName: ${file_name}  
 * @copyright_author: jitwxs
 * @date: ${date} ${time} 
 */</template><template autoinsert="true" context="overridecomment_context" deleted="false" description="Comment for overriding methods" enabled="true" id="org.eclipse.jdt.ui.text.codetemplates.overridecomment" name="overridecomment">/* (non-Javadoc)
 * ${see_to_overridden}
 */</template><template autoinsert="false" context="typecomment_context" deleted="false" description="Comment for created types" enabled="true" id="org.eclipse.jdt.ui.text.codetemplates.typecomment" name="typecomment">/** 
 * @className ${file_name}
 * @author jitwxs
 * @date ${date} ${time} 
*/ </template><template autoinsert="true" context="settercomment_context" deleted="false" description="Comment for setter method" enabled="true" id="org.eclipse.jdt.ui.text.codetemplates.settercomment" name="settercomment">/**
 * @param ${param} the ${bare_field_name} to set
 */</template><template autoinsert="false" context="methodcomment_context" deleted="false" description="Comment for non-overriding methods" enabled="true" id="org.eclipse.jdt.ui.text.codetemplates.methodcomment" name="methodcomment">/**
 * @author jitwxs
 * @date ${date} ${time} 
 * ${tags}
*/ </template></templates>
```
