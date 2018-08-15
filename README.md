# addPermissionAccountSystem
在dva示例项目AccountSystem增加权限管理的改造，learning react dva


本项目代码来自于dva示例项目AccountSystem,只是把原项目首页menu的显示方式改为根据登录用户的权限显示menu方式；
以下为修改的steps:
step1:将原来的src/component/Header组件中的menus，manageChildMenus，billsChildMenus置为空[];
step2:显示该dumb组件的smart组件为src/routes/HomePage/HomePage.jsx,改为

const HomePage = ({ children, home, systemUser }) => {
    const { activeIndex } = home;
    const { permissionInfo } = systemUser;
    const height = window.innerHeight - 64;
    return (
        <div className={homePage}>
            <Header activeIndex={activeIndex} menus={permissionInfo.menus} manageChildMenus={permissionInfo.manageChildMenus} billsChildMenus={permissionInfo.billsChildMenus} />
            <SystemInfo />
            <div className={container} style={{ height }}>
                {children}
            </div>
        </div>
    );
};

function mapStateToProps({ home, systemUser }) {
    return { home, systemUser };
}
增加systemUser的model，因为permission的数据是从登录时传入的；
step3：修改/src/models/systemUser,
    3.1:在state中添加permission参数：
state: {
        username: '',
        isLogin: false,
        modalVisible: false,
        authToken: '',
        pathname: '/',
        logupModalVisible: false,
        permissionInfo: { menus: [], manageChildMenus: [], billsChildMenus: [] },
        // menus: [],
        // manageChildMenus: [],
        // billsChildMenus: []
    },
    3.2:subscription中添加permissionInfo:
    subscriptions: {
        setup({ dispatch, history }) {
            history.listen(location => {
                if (location.pathname == '/') {
                    //权限验证通过
                    if (sessionStorage.getItem('userInfo') && sessionStorage.getItem('permissionInfo')) {
                        dispatch({
                            type: 'loginSuccess',
                            payload: { userInfo: JSON.parse(sessionStorage.getItem('userInfo')), permissionInfo: JSON.parse(sessionStorage.getItem('permissionInfo')) } || {}
                        });
                    }
                }
            });
        },
    },
  3.3:effects中登录和登出时添加permissionInfo 如下：
  *doLogin({ payload }, { call, put }) {
            let {
                userData,
                resolve,
                reject
            } = payload;
            yield put({ type: 'showLoading' });
            const { data } = yield call(doLogin, userData);
            if (data && data.success) {
                let userInfo = data.userInfo;
                let permissionInfo = data.permissionInfo;
                yield sessionStorage.setItem('userInfo', JSON.stringify(userInfo));
                yield sessionStorage.setItem('permissionInfo', JSON.stringify(permissionInfo));
                //登录成功
                yield put({
                    type: 'loginSuccess',
                    payload: { userInfo, permissionInfo }
                });
                resolve();
            } else {
                reject(data);
            }
        },
        *doLogout({ payload }, { call, put }) {
            const { data } = yield call(doLogout);
            if (data && data.success) {
                yield sessionStorage.removeItem('userInfo')
                yield sessionStorage.removeItem('permissionInfo')
                //退出登录成功
                yield put({
                    type: 'logoutSuccess',
                    payload: { userInfo: data.userInfo, permissionInfo: { menus: [], manageChildMenus: [], billsChildMenus: [] } }
                });
            }
        }
        3.4：最后在reducer登入登出中添加permissionInfo 如下：
        loginSuccess(state, action) {
            let userInfo = action.payload.userInfo;
            let permissionInfo = action.payload.permissionInfo;
            return {
                ...state, ...userInfo, permissionInfo, isLogin: true, modalVisible: false
            };
        },
        logoutSuccess(state, action) {
            let permissionInfo = action.payload.permissionInfo;
            return { ...state, username: '', permissionInfo, authToken: '', isLogin: false };
        },
        
 这样前端的代码就修改完成了，主要的逻辑就是，首先在systemUser的登入登出中添加permissionInfo的状态，从服务端获取,然后在smart组件HomePage中，将systemUser的state中的permissionInfo以props的方式传入到子组件Header中，最后在dumb组件Header中只要通过this.props.menu这些属性就可以根据不同登录的用户具有的不同权限，渲染出menus；这里只列出了前端的修改代码，后端的逻辑按普通的权限管理逻辑，当前端通过effects调取接口时，传入permissionInfo即可。
 本文的原始项目代码为dva提供的示例项目AccountSystem.
