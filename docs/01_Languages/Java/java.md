---
layout: default
title: Java
nav_order: 10
has_children: true
permalink: docs/01_Languages/Java
parent: Languages
---

# Java
{: .no_toc }

1. TOC
{:toc}

자바 소스 코드 변환 과정



소스 코드 작성 -? .java 파일

컴파일 // javac 가 자바 소스 코드를 바이트 코드로 변환합니다. 해당 코드는 .class 파일에 저장됩니다.

클래스 로더 // 바이트 코드가  JVM으로 로드되기 전에 클래스 로더에 의해 필요한 클래스를 로드하고 메모리에 적재하는 과정을 수행합니다.

JVM에 의해 바이트 코드가 메모리에 적재된 후, 실행되면서 자바 프로그램은 프로세스가 됩니다. 이때 런타임 과정이 발생한다. 프로세스 이후로 부터 JVM에 의해 바이트 코드가 메모리에 적재된 후를 런타임 과정으로 본다.

런타임이란 프로그램이 실제로 실행되는 시점을 의미한다.

런타임에서 수행되는 작업

클래스 로딩 : 앞서 언급한 클래스 로더에 의해 클래스들이 메모리에 적재되는 과정


# containsKey , containsValue

containsKey(Object key) : Map 객체에 특정 키가 포함되었는지 확인한다. 반환값은 boolean

containsValue(Object value) : Map 객체에 특정 값이 포함되었는지 확인. 반환값은 boolean

put - 맵 객체에 키-값 쌍을 저장합니다.
get - 키 값을 반환한다. 매개변수로 키 값 주입
size - map 또는 collection의 요소 개수를 반환

Annotation - 메타데이터를 제공하기 위해 사용되며, 주로 컴파일러에게 추가 정보를 전달하거나 런타임 시 특정 동작 제어

예시로 @Override는 메서드가 상위클래스의 메서드를 재정의 하고있음을 나타냄
@Deprecated, SuppressWarnings

Generics - 클래스 내부에서 사용할 데이터 타입을 외부에서 지정할 수 있는 기법 / 코드의 안정성과 재사용성을 높여준다.

제네릭을 사용하면 따로 형변환을 하지 않아도 된다.