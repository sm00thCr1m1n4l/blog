# 在typescript中使用 redux connect

```jsx
import * as React from 'react'
React.createElement
import { connect } from 'react-redux'
import { ThunkDispatch} from 'redux-thunk'
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

export type IHOCProps = {
  myStore: {
    data: any[]
    action: () => void
  }
}
export function withSomeStore<BaseProps>(Cmp: React.ComponentClass<BaseProps & IHOCProps>) {
  type HOCRecivedProps = IMapStateToProps & IMapDispatchToProps
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

