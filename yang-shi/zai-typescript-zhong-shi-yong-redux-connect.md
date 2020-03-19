# 在typescript中使用HOC来包装 redux connect

```jsx
import * as React from 'react'
import { connect } from 'react-redux'
interface IMapStateToProps {
  someData: any[]
}

interface IMapDispatchToProps {
  myAction: () => void
}

const mapStateToProps = (state: any): IMapStateToProps => {
  return {
    someData: state.someData
  }
}

const mapDispatchToProps = (): IMapDispatchToProps => {
  return {
    myAction: () => { }
  }
}

export type WrappedRecivedProps = {
  myStore: {
    data: any[],
    action: () => void
  }
}
type HOCRecivedProps = IMapStateToProps & IMapDispatchToProps
export function withSomeStore<BaseProps>(Cmp: React.ComponentClass<BaseProps & WrappedRecivedProps>) {
  class SomeStore extends React.Component<HOCRecivedProps> {
    public render() {
      const { someData, myAction, ...props } = this.props
      return (
        <Cmp myStore={{ data: someData, action: myAction }}  {...(props as BaseProps)} />
      )
    }
  }
  return connect<
    IMapStateToProps,
    IMapDispatchToProps,
    BaseProps
  >(mapStateToProps, mapDispatchToProps)(SomeStore)
}

```

