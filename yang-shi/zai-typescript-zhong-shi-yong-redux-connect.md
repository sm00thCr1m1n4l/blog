# 在typescript中使用 redux connect

```jsx
import * as React from 'react'
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

export type IBlogStoreHOC = {
  myStore: {
    data: any[]
    action: () => void
  }
}
export function withBlogStore<BaseProps>(Cmp: React.ComponentClass<BaseProps & IBlogStoreHOC>) {
  type HOCRecivedProps = IMapStateToProps & IMapDispatchToProps
  class BlogStore extends React.Component<HOCRecivedProps> {
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
  >(mapStateToProps, mapDispatchToProps)(BlogStore)
}

```

