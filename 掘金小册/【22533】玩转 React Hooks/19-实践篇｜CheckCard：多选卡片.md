通过上两节的内容，我们知道 Form 组件是如何制作，又是如何控制表单控件的，那么这节我们自己实现一个简单的表单控件，通过这个组件，更好地理解 Hooks
在组件中的应用，如何配合 Form 组件联合使用。

在这节，我们去实现 ProComponents 中
[CheckCard](https://procomponents.ant.design/components/check-card) 组件。

之所以选择这个组件，是因为它与 Form 组件的结构类似，其次它的实现相对简单，在了解它的同时，也能巩固之前的 Form 组件，所以是个不错的示例组件。

**CheckCard：** 多选卡片，用于集合多种相关说明信息，并且可以被选择，用在 Form
表单中，成为一个效果非常好的表单控件。它分为两部分，分别是：

  1. **CheckCard：** 用于展示头像、标题、描述信息等，具备选中、禁用、加载等状态，可单独使用。
  2. **CheckCard.Group：** 集中控制 CheckCard，使其受控，可配合 Form 组件联合使用。

接下来，我们依次来实现它们。

# CheckCard

我们先不用考虑 CheckCard.Group 的实现，先去实现 CheckCard，再来实现 CheckCard.Group。

## 1\. 基本布局

在 CheckCard 中具备四种布局元素，分别是 avatar（头像）、
title（标题）、description（描述信息）、extra（右上角额外信息）。这里用 useCss 来简单实现 CheckCard 的样式即可：

    
    
    const CheckCard = (props: CheckCardProps) => {
    
      const dataMemo = useCreation(() => {
        const avatarDom = avatar ? (
          <div className={styleDateMemo["check-card-avatar"]}>
            {typeof avatar === "string" ? (
              <Avatar size={48} shape="square" src={avatar} />
            ) : (
              avatar
            )}
          </div>
        ) : null;
    
        const header = (title ?? extra) !== null && (
          <div className={styleDateMemo["check-card-header"]}>
            <div className={styleDateMemo["check-card-title"]}>{title}</div>
            {extra && (
              <div className={styleDateMemo["check-card-extra"]}>{extra}</div>
            )}
          </div>
        );
    
        const descriptionDom = description ? (
          <div className={styleDateMemo["check-card-description"]}>
            {description}
          </div>
        ) : null;
    
        return (
          <div className={styleDateMemo["check-card-content"]}>
            {avatarDom}
            {header || descriptionDom ? (
              <div className={styleDateMemo["check-card-detail"]}>
                {header}
                {descriptionDom}
              </div>
            ) : null}
          </div>
        );
      }, [title, extra, description]);
      
      return (
        <div>
          {dataMemo}
        </div>
      );
    }
    

**效果：**

![image.png](img\19\1.image)

## 2\. 额外信息

可以通过 extra 来制作卡片的额外操作，但要注意，我们在整个卡片都附有点击事件，所以我们的额外操作中一定要 **阻止事件冒泡** ，即
`e.stopPropagation();`。

![img2.gif](img\19\2.image)

## 3\. 基本状态的改变

在 CheckCard 中，共有三种状态，分别是未选中、选中、禁用，而这三种状态所对应的样式都有所改变，此时我们可以利用 `classNames`
来帮助我们处理卡片的样式，使效果更美观。

    
    
    const CheckCard = (props: CheckCardProps) => {
      const {
        avatar,
        title,
        extra,
        description,
        disabled = false,
        loading = false,
        style = {},
        ...params
      } = props;
    
      const [checked, setChecked] = useSafeState<boolean>(
        params.defaultChecked || false
      );
    
      const styleClassName: StylesBooleanProps = {};
      styleClassName[useCss(styles["check-card"])] = true;
      styleClassName[useCss(styles["check-card-checked"])] = !!checked;
      styleClassName[useCss(styles["check-card-disabled"])] = !!disabled;
      styleClassName[useCss(styles["check-card-disabled-after"])] = !!checked && !!disabled;
    
      return (
        <div
          className={classNames(styleClassName)}
          style={style}
          onClick={(v) => {
            if (!disabled && !loading) {
              params.onClick && params.onClick(v);
              params.onChange && params.onChange(!checked);
              setChecked((v) => !v);
            }
          }}
        >
          {dataMemo}
        </div>
      );
    };
    

**效果：**

![](img\19\3.image)

> 其中，鼠标移动到卡片上可以通过 hover 属性，右上角的标可以通过 after 简单制作，鼠标的样式可以通过 cursor 来控制。

## 4\. 加载状态

通过配置 loading 属性可以配置组件的加载状态，可以通过 Row、Col 来做简单的布局，然后通过 `linear-gradient`
来控制颜色的渐变，再配合 `animation` 控制颜色的滚动。

    
    
    const Loading = () => {
      return (
        <div className={useCss(styles["check-card-loading-content"])}>
          <Row gutter={8}>
            <Col span={22}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
          </Row>
          <Row gutter={8}>
            <Col span={8}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
            <Col span={14}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
          </Row>
          <Row gutter={8}>
            <Col span={6}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
            <Col span={16}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
          </Row>
          <Row gutter={8}>
            <Col span={13}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
            <Col span={9}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
          </Row>
          <Row gutter={8}>
            <Col span={4}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
            <Col span={3}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
            <Col span={14}>
              <div className={useCss(styles["check-card-loading"])} />
            </Col>
          </Row>
        </div>
      );
    };
    

**效果：**

![](img\19\4.image)

# CheckCard.Group

**CheckCard.Group：** 布局组件，用来集中控制 CheckCard，通过 value 和 onChange
使其受控，也可通过其他属性来整体控制 CheckCard，配合 Form 组件联合使用。

## 1\. 数据传递

CheckCard.Group 和 CheckCard 组件存在深层的嵌套关系，所以需要通过 `context（ createContext +
useContext ）`跨层级方式传递数据。

    
    
      // GroupContext
      import { createContext } from "react";
      import { SelectGroupConnextType } from "./interface.d";
    
      const GroupContext = createContext<SelectGroupConnextType | null>(null);
    
      export default GroupContext;
      
      // Group
      const Group: React.FC<GroupProps> = (props) => {
        ...
        <GroupContext.Provider
          value={{ ... }}
        >
          <div className={useCss(styles["select-card-group"])} style={style}>
            {params.children}
          </div>
        </GroupContext.Provider>
      }
    

## 2\. 注册与卸载

因为我们需要 CheckCard.Group 去管理 CheckCard 的数据，所以 CheckCard.Group 要监听 CheckCard
的数据变化，换言之，CheckCard 要进行注册和卸载，而判断的依据则是 value。

同时，在 CheckCard.Group 中通过 `new Map()` 集中管理数据，并以 useRef 保存数据源，防止闭包。

    
    
      // Group
      const Group: React.FC<GroupProps> = (props) => {
        const ref = useRef<Map<ValueType, any>>(new Map());
    
        // 注册
        const registerValue = (value: string) => {
          ref.current?.set(value, true);
        };
    
        // 卸载
        const cancelValue = (value: string) => {
          ref.current?.delete(value);
        };
        
        ...
      }
      
      // index
      const CheckCard = (props: CheckCardProps) => {
        useEffect(() => {
          params.value && group?.registerValue?.(params.value);
          return () => {
            params.value && group?.cancelValue?.(params.value);
          };
        }, [params.value]);
        
        ...
      }
    

## 3\. 使 CheckCard 受控

如果存在多个 CheckCard，那么它们每一个都是独立的组件，但如果在 CheckCard.Group 下，CheckCard 则需要受
CheckCard.Group 的控制，由 CheckCard.Group 的 value 控制所有的 CheckCard。

那么 value 将会存在三种形式：

  1. **undefined：** value 不存在时；
  2. **string：** 字符串，单选时；
  3. **string[]：** 数组，多选时。

触发 CheckCard.Group 的变化时机则是 CheckCard 的 onChange 方法。如：

    
    
      // Group
      const Group: React.FC<GroupProps> = (props) => {
        const { multiple = false, onChange, ...params } = props;
        const [stateValue, setStateValue] = useSafeState<GroupValueType>();
        
        ....
        const selectOption = (option: SelectOptionProps) => {
          if (multiple) {
            let newValue: ValueType[] = [];
            const stateValues = stateValue as ValueType[];
            const flag = stateValues?.includes(option.value);
            newValue = [...(stateValues || [])];
            if (flag) {
              newValue = newValue.filter((itemValue) => itemValue !== option.value);
            } else {
              newValue.push(option.value);
            }
    
            setStateValue?.(newValue);
            onChange && onChange(newValue);
          } else {
            let newValue = stateValue;
            if (newValue === option.value) {
              newValue = undefined;
            } else {
              newValue = option.value;
            }
            setStateValue?.(newValue);
            onChange && onChange(newValue);
          }
        };
        
        ...
      }
      
      // index
      const CheckCard = (props: CheckCardProps) => {
        const selectData: any = {};
         
         ...
         selectData.checked = checked;
         if (group) { // 通过 Group 组件控制对应的选中状态
          const isChecked = group.multiple
            ? group.value?.includes(params.value)
            : group.value === params.value;
          selectData.checked = isChecked;
        }
    
        return (
          <div
            className={classNames(styleClassName)}
            style={style}
            onClick={(v) => {
              if (!disabled && !loading) {
                ...
                group?.selectOption?.({ value: props.value });
              }
            }}
          >
            {dataMemo}
          </div>
        );
      }
    

**效果：**

![img3.gif](img\19\5.image)

## 4\. 配合 Form 组件使用

在上两节的学习中，我们知道 Form.Item 控制表单控件是通过 `React.cloneElement` 的帮助，给对应的表单控件加入 value 和
onChange 元素。也就是说，要想自定义控件跟 Form 绑定关系，只需要存在 value 和 onChange 这两个属性，使其受控配合即可。

因为 Form 组件会统一管理 value，所以在 CheckCard.Group 中要对 value 进行监控，控制 value 属性。

    
    
      // Check.Group
      const Group: React.FC<GroupProps> = (props) => {
      
        const [stateValue, setStateValue] = useSafeState<GroupValueType>();
        
        useEffect(() => {
          setStateValue(params.value || params.initValue);
        }, [params.value]);
        
        ...
      }
    

**代码演示：**

    
    
    import React from "react";
    import CheckCard from "./CheckCard";
    import { Button, message } from "antd";
    import Form from "../Form/HooksForm";
    
    const Index: React.FC = () => {
      return (
        <>
          <h1>在 Form 表单的应用</h1>
          <Form
            initialValues={{ card: "A" }}
            onFinish={(data: any) => {
              console.log("表单数据:", data);
            }}
            onReset={() => {
              console.log("重制表单成功");
            }}
          >
            <Form.Item label="选择卡片-单选" name="card" styles={{ with: "100%" }}>
              <CheckCard.Group>
                <CheckCard title="Card A" description="一起玩转Hooks吧" value="A" />
                <CheckCard title="Card B" description="一起玩转Hooks吧" value="B" />
                <CheckCard title="Card C" description="一起玩转Hooks吧" value="C" />
              </CheckCard.Group>
            </Form.Item>
            <Form.Item label="选择卡片-多选" name="card-multiple">
              <CheckCard.Group multiple>
                <CheckCard title="Card A" description="一起玩转Hooks吧" value="A" />
                <CheckCard title="Card B" description="一起玩转Hooks吧" value="B" />
                <CheckCard title="Card C" description="一起玩转Hooks吧" value="C" />
              </CheckCard.Group>
            </Form.Item>
            <Form.Item>
              <Button type="primary" htmlType="submit">
                提交
              </Button>
              <Button style={{ marginLeft: 4 }} htmlType="reset">
                重置
              </Button>
            </Form.Item>
          </Form>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![img4.gif](img\19\6.image)

## 5\. 集中控制 loading

CheckGroup.Card 除了可以控制 CheckGroup 的 value 外，还可以集中控制加载状态、边框样式、卡片大小等，原理与 value
一样。这里巩固一下，加一个 loading 状态，整体去控制 CheckGroup。

    
    
      const CheckCard = (props: CheckCardProps) => {
        const selectData: any = {};
         
        selectData.checked = checked;
        selectData.loading = loading;
        if (group) {
          // 通过 Group 组件控制对应的选中状态
          const isChecked = group.multiple
            ? group.value?.includes(params.value)
            : group.value === params.value;
          selectData.checked = isChecked;
          selectData.loading = loading || group.loading;
        }
        
        // 之后使用 loading 的地方都换成 selectData.loading 即可
        ...
      }
      
      // 使用
      <h1>集中控制 Loading：</h1>
      <CheckCard.Group loading>
        <CheckCard title="Card A" description="一起玩转Hooks吧" value="A" />
        <CheckCard title="Card B" description="一起玩转Hooks吧" value="B" />
        <CheckCard title="Card C" description="一起玩转Hooks吧" value="C" />     
      </CheckCard.Group>
    

**效果：**

![image.png](img\19\7.image)

# 小结

通过本小节的阅读，我们知道 CheckCard 组件的实现原理，了解到 Hooks 在组件中的应用，简单总结下：

  1. 明确数据流、数据走向，确保数据的传递。任何组件，首先要确保数据的流向，规划好数据源，确保数据由单个地方产生，一层一层传递，不要有多个数据源。
  2. 组件传递需要考虑 **多层级嵌套** 的关系，优先使用 `context（ createContext + useContext ）`跨层级方式完成。
  3. 有注册，就要有卸载，确保数据的唯一、整洁，防止未卸载的数据影响组件本身，造成内存泄漏。

以上就是 CheckCard 组件的制作流程。

