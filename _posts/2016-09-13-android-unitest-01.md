---
layout: post
title:  "构建高效的Android单元测试(一)"
date:   2016-09-13 21:35:31 +0800
categories: unit test
---

单元测试是app开发中的基本测试策略，编写和运行单元测试可以快速的验证程序单元的正确性。代码有变更或者发版之前都要把单元测试跑一遍，以便发现和解决由于更改代码而引入的问题。

Android中的单元测试一般测试一个很小的代码单元，可能是一个方法，一个类或者一个组件。编写单元测试的目的就是测试这些代码单元的逻辑正确性。一般情况下每个测试单元比较独立的，不需要依赖或者影响其他测试单元。

在Android中可能会涉及到Android的一些特有的类或者组件的，比如SharedPreferences，Activity等，如果在不安装到Android的模拟器或者设备上就能测试的话，就要借助于mock框架，创建模拟的Android特有的组件以供测试期间使用。

一般情况下测试Android的程序，需要创建两种类型的自动化的单元测试：

* 本地测试：所谓的本地测试就是你写的单元测试代码不需要运行在模拟器或者物理设备上，直接在你的开发机上编译运行。这种方式不需要依赖Android的Framework，或者依赖的组件或者类似可以通过mock的方式进行测试。

* Instrumented tests: 这种类型的单元测试运行在模拟器或者物理设备上，这些测试需要访问比如Context等类。这种测试必须依赖Anroid，并且依赖的对象不方便mock。


### 构建本地单元测试

本地单元测试使用场景：不依赖或者对Android的依赖很少。

本地单元测试的好处就是高效，不需要把测试代码放在android机器上运行，减少了编译和和执行的时间。

但是在依赖部分Android的组件的时候，需要通过mock框架进行弥补，Android官方推荐是用的是Mockito。

#### 测试环境

标准的Android Studio Project中，你必须把本地单元测试代码放在如下目录：module-name/src/test/java/

当你创建新的Project的时候，Android Studio已经帮你创建好了。不过也可以自己手工创建这个目录，比较麻烦，还是让工具帮忙吧。

有了单元测试的目录之后，还需要配置单元测试需要的一些依赖，比如单元测试标准框架JUnit4,如果需要和Android的依赖进行交互的话，还需要依赖mock框架Mockito。

在你的module的根目录的build.gradle里面声明如下依赖：

```java
dependencies {
    // Required -- JUnit 4 framework
    testCompile 'junit:junit:4.12'
    // Optional -- Mockito framework
    testCompile 'org.mockito:mockito-core:1.10.19'
}
```
比如在官方提供的单元测试demo中，是在app module中的build.gradle文件。

#### 创建本地单元测试类

>本地单元测试的例子来自google在github上的demo: https://github.com/googlesamples/android-testing

>本地单元测试在BasicSample这个project中：https://github.com/googlesamples/android-testing/tree/master/unit/BasicSample

下面贴出来 BasicSample/app/src/test/java中的代码

com.example.android.testing.unittesting.BasicSample

```java
package com.example.android.testing.unittesting.BasicSample;

import android.test.suitebuilder.annotation.SmallTest;

import org.junit.Test;

import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertTrue;


/**
 * Unit tests for the EmailValidator logic.
 */
@SmallTest
public class EmailValidatorTest {


    @Test
    public void emailValidator_CorrectEmailSimple_ReturnsTrue() {
        assertTrue(EmailValidator.isValidEmail("name@email.com"));
    }

    @Test
    public void emailValidator_CorrectEmailSubDomain_ReturnsTrue() {
        assertTrue(EmailValidator.isValidEmail("name@email.co.uk"));
    }

    @Test
    public void emailValidator_InvalidEmailNoTld_ReturnsFalse() {
        assertFalse(EmailValidator.isValidEmail("name@email"));
    }

    @Test
    public void emailValidator_InvalidEmailDoubleDot_ReturnsFalse() {
        assertFalse(EmailValidator.isValidEmail("name@email..com"));
    }

    @Test
    public void emailValidator_InvalidEmailNoUsername_ReturnsFalse() {
        assertFalse(EmailValidator.isValidEmail("@email.com"));
    }

    @Test
    public void emailValidator_EmptyString_ReturnsFalse() {
        assertFalse(EmailValidator.isValidEmail(""));
    }

    @Test
    public void emailValidator_NullEmail_ReturnsFalse() {
        assertFalse(EmailValidator.isValidEmail(null));
    }
}

```

依赖的是JUnit4,比JUnit3要方便很多。只需要在需要测试的方法前面使用@Test注解即可。对测试的结果进行断言，需要使用org.junit.Assert类，上述例子中，为了使用方便使用了静态引用的语法，直接使用Assert方法的assertFalse和assertTrue方法，显得更加的便捷。

#### 如果执行单元测试？

（1）在Android Studio中，选中测试的单元测试方法，点右键可以【Run "email...()"】可以单独执行选中的单元测试，执行成功之后会显示成功的绿条。

![在Android Studio中运行单元测试]({{ site.url }}/image/android-img-01-001.png)

其中也可以右键运行【Debug "emial...()"】选项，可以对单元测试进行Debug，方便调试。

右键运行【Run "email...()" with Coverage】选项，可以把此单元测试类的所有方法执行一遍。

还有一种方式：选择测试目录，然后右键运行也可以，会运行这个文件夹里的单元测试方法。

（2）是用命令执行单元测试

  到工程的根目录下执行：
  	
  	./gradlew test

  生成报告路径：

	HTML测试结果: BasicSample/app/build/reports/tests/

	XML测试结果: BasicSample/app/build/test-results/
	
### Mock Android 依赖的单元测试

上面的单元测试是不依赖任何Android的api的，所以编写和编译起来都比较简单和高效。当单元测试依赖了部分Android的api的时候，将如何进行单元测试呢？mock框架就是解决这个问题的。

#### Mockito

Mockito是Android官方建议使用的mocking框架。Mockito为单元测试提供Android系统相关的mock对象或者数据，保证单元测试的正常进行。

如何配置Mockito前面已经讲解过，这里就不在讲解。

#### 如何使用

为了更加直观，官方文档中代码如下：

```java
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.CoreMatchers.*;
import static org.mockito.Mockito.*;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.Mock;
import org.mockito.runners.MockitoJUnitRunner;
import android.content.SharedPreferences;

@RunWith(MockitoJUnitRunner.class)
public class UnitTestSample {

    private static final String FAKE_STRING = "HELLO WORLD";

    @Mock
    Context mMockContext;

    @Test
    public void readStringFromContext_LocalizedString() {
        // Given a mocked Context injected into the object under test...
        when(mMockContext.getString(R.string.hello_word))
                .thenReturn(FAKE_STRING);
        ClassUnderTest myObjectUnderTest = new ClassUnderTest(mMockContext);

        // ...when the string is returned from the object under test...
        String result = myObjectUnderTest.getHelloWorldString();

        // ...then the result should be the expected one.
        assertThat(result, is(FAKE_STRING));
    }
}
```

##### @RunWith(MockitoJUnitRunner.class)

@RunWith 注解的意思是，这个单元测试是基于MockitoJUnitRunner的测试类，需要mock Android api相关的对象。

##### @Mock 

@Mock注解修饰的变量，代表需要Mock框架忙创建此对象，例子中是创建Context对象

#### org.mockito.Mockito.when()

when()方法可以根据条件，thenReturn()返回相应的值。

#### 官方Demo中的例子

官方BasicSimple中的验证邮箱，存储用户信息到SharedPreferences的例子。

```java
/*
 * Copyright 2015, The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package com.example.android.testing.unittesting.BasicSample;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.CoreMatchers.*;
import static org.mockito.Mockito.*;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.Mock;
import org.mockito.runners.MockitoJUnitRunner;

import android.content.SharedPreferences;
import android.test.suitebuilder.annotation.SmallTest;

import java.util.Calendar;


/**
 * Unit tests for the {@link SharedPreferencesHelper} that mocks {@link SharedPreferences}.
 */
@SmallTest
@RunWith(MockitoJUnitRunner.class)
public class SharedPreferencesHelperTest {

    private static final String TEST_NAME = "Test name";

    private static final String TEST_EMAIL = "test@email.com";

    private static final Calendar TEST_DATE_OF_BIRTH = Calendar.getInstance();

    static {
        TEST_DATE_OF_BIRTH.set(1980, 1, 1);
    }

    private SharedPreferenceEntry mSharedPreferenceEntry;

    private SharedPreferencesHelper mMockSharedPreferencesHelper;

    private SharedPreferencesHelper mMockBrokenSharedPreferencesHelper;

    @Mock
    SharedPreferences mMockSharedPreferences;

    @Mock
    SharedPreferences mMockBrokenSharedPreferences;

    @Mock
    SharedPreferences.Editor mMockEditor;

    @Mock
    SharedPreferences.Editor mMockBrokenEditor;

    @Before
    public void initMocks() {
        // Create SharedPreferenceEntry to persist.
        mSharedPreferenceEntry = new SharedPreferenceEntry(TEST_NAME, TEST_DATE_OF_BIRTH,
                TEST_EMAIL);

        // Create a mocked SharedPreferences.
        mMockSharedPreferencesHelper = createMockSharedPreference();

        // Create a mocked SharedPreferences that fails at saving data.
        mMockBrokenSharedPreferencesHelper = createBrokenMockSharedPreference();
    }

    @Test
    public void sharedPreferencesHelper_SaveAndReadPersonalInformation() {
        // Save the personal information to SharedPreferences
        boolean success = mMockSharedPreferencesHelper.savePersonalInfo(mSharedPreferenceEntry);

        assertThat("Checking that SharedPreferenceEntry.save... returns true",
                success, is(true));

        // Read personal information from SharedPreferences
        SharedPreferenceEntry savedSharedPreferenceEntry =
                mMockSharedPreferencesHelper.getPersonalInfo();

        // Make sure both written and retrieved personal information are equal.
        assertThat("Checking that SharedPreferenceEntry.name has been persisted and read correctly",
                mSharedPreferenceEntry.getName(),
                is(equalTo(savedSharedPreferenceEntry.getName())));
        assertThat("Checking that SharedPreferenceEntry.dateOfBirth has been persisted and read "
                + "correctly",
                mSharedPreferenceEntry.getDateOfBirth(),
                is(equalTo(savedSharedPreferenceEntry.getDateOfBirth())));
        assertThat("Checking that SharedPreferenceEntry.email has been persisted and read "
                + "correctly",
                mSharedPreferenceEntry.getEmail(),
                is(equalTo(savedSharedPreferenceEntry.getEmail())));
    }

    @Test
    public void sharedPreferencesHelper_SavePersonalInformationFailed_ReturnsFalse() {
        // Read personal information from a broken SharedPreferencesHelper
        boolean success =
                mMockBrokenSharedPreferencesHelper.savePersonalInfo(mSharedPreferenceEntry);
        assertThat("Makes sure writing to a broken SharedPreferencesHelper returns false", success,
                is(false));
    }

    /**
     * Creates a mocked SharedPreferences.
     */
    private SharedPreferencesHelper createMockSharedPreference() {
        // Mocking reading the SharedPreferences as if mMockSharedPreferences was previously written
        // correctly.
        when(mMockSharedPreferences.getString(eq(SharedPreferencesHelper.KEY_NAME), anyString()))
                .thenReturn(mSharedPreferenceEntry.getName());
        when(mMockSharedPreferences.getString(eq(SharedPreferencesHelper.KEY_EMAIL), anyString()))
                .thenReturn(mSharedPreferenceEntry.getEmail());
        when(mMockSharedPreferences.getLong(eq(SharedPreferencesHelper.KEY_DOB), anyLong()))
                .thenReturn(mSharedPreferenceEntry.getDateOfBirth().getTimeInMillis());

        // Mocking a successful commit.
        when(mMockEditor.commit()).thenReturn(true);

        // Return the MockEditor when requesting it.
        when(mMockSharedPreferences.edit()).thenReturn(mMockEditor);
        return new SharedPreferencesHelper(mMockSharedPreferences);
    }

    /**
     * Creates a mocked SharedPreferences that fails when writing.
     */
    private SharedPreferencesHelper createBrokenMockSharedPreference() {
        // Mocking a commit that fails.
        when(mMockBrokenEditor.commit()).thenReturn(false);

        // Return the broken MockEditor when requesting it.
        when(mMockBrokenSharedPreferences.edit()).thenReturn(mMockBrokenEditor);
        return new SharedPreferencesHelper(mMockBrokenSharedPreferences);
    }
}


```

学习上面的这些例子，只能基本了Android单元测试的基本模式，以及一些基本的测试原理和思想。这些还不够，需要在实践中进行深入的研究摸索。会编写单元测试只是一个开始，深入理解单元测试的原理，以及如何把单元测试应用到项目的工程化当中，将会是一个有意思并且很具有挑战性的事情。

后面还会继续把官方Demo一一测试和讲解完。然后再去进行自己的单元测试定制化。

参考文献：

[1]Android 官方文档
