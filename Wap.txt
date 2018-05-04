package com.jydp.controller.wap;

import com.alibaba.fastjson.JSONObject;
import com.iqmkj.utils.Base64Util;
import com.iqmkj.utils.MD5Util;
import com.iqmkj.utils.StringUtil;
import com.jydp.entity.BO.JsonObjectBO;
import com.jydp.entity.BO.UserSessionBO;
import com.jydp.entity.DO.transfer.JydpCoinConfigDO;
import com.jydp.entity.DO.user.UserCurrencyNumDO;
import com.jydp.entity.DO.user.UserDO;
import com.jydp.entity.VO.UserCoinConfigVO;
import com.jydp.interceptor.UserWebInterceptor;
import com.jydp.service.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

import javax.servlet.http.HttpServletRequest;
import java.util.List;

/**
 * 用户币种提现
 * @Author: wqq
 */
@Controller
@RequestMapping("/userWap/userCoinWithdrawal")
@Scope(value = "prototype")
public class WapUserCoinWithdrawalController {

    /**  JYDP币种转出管理 */
    @Autowired
    private IJydpCoinConfigService jydpCoinConfigService;

    /**  用户账号 */
    @Autowired
    private IUserService userService;

    /**  盛源链账号绑定表 */
    @Autowired
    private ISylUserBoundService sylUserBoundService;

    /** 用户币数量 */
    @Autowired
    private IUserCurrencyNumService userCurrencyNumService;

    /** 系统手机验证 */
    @Autowired
    private ISystemValidatePhoneService systemValidatePhoneService;

    /** JYDP用户币种转出记录 */
    @Autowired
    private  IJydpUserCoinOutRecordService jydpUserCoinOutRecordService;

    /**  币种提现页面展示 */
    @RequestMapping("show.htm")
    public String show(HttpServletRequest request) {
        UserSessionBO userBo = UserWebInterceptor.getUser(request);
        if (userBo == null) {
            request.setAttribute("code", 4);
            request.setAttribute("message", "未登录！");
            return "page/web/login";
        }

        UserDO user = userService.getUserByUserId(userBo.getUserId());
        if (user == null) {
            request.setAttribute("code", 3);
            request.setAttribute("message", "改用户不存在");
            return "page/web/login";
        }
        if (user.getAccountStatus() != 1) {
            request.setAttribute("code", 3);
            request.setAttribute("message", "该账号已被禁用");
            return "page/web/login";
        }

        //用户所有币种,数量及转出管理
        List<UserCoinConfigVO> userCoinConfigList = jydpCoinConfigService.listUserCoinConfigByUserId(userBo.getUserId());

        String phoneNumber = user.getPhoneNumber();
        String phoneNumberEn = null;
        if (phoneNumber != null && phoneNumber.length() > 5) {
            phoneNumberEn = phoneNumber.substring(0, 3) + "****" + phoneNumber.substring(phoneNumber.length() - 4, phoneNumber.length());
        }

        request.setAttribute("userCoinConfigList",userCoinConfigList);
        request.setAttribute("phoneAreaCode",user.getPhoneAreaCode());
        request.setAttribute("phoneNumber",phoneNumber);
        request.setAttribute("phoneNumberEn",phoneNumberEn);

        return "page/wap/coinWithdrawal";
    }

    /**  获取币种管理信息 */
    @RequestMapping(value = "coinConfig.htm")
    public @ResponseBody
    JsonObjectBO coinConfig(HttpServletRequest request) {
        JsonObjectBO response = new JsonObjectBO();
        UserSessionBO userBo = UserWebInterceptor.getUser(request);
        if (userBo == null) {
            response.setCode(4);
            response.setMessage("未登录");
            return response;
        }

        String currencyIdStr = StringUtil.stringNullHandle(request.getParameter("currencyId"));
        if (!StringUtil.isNotNull(currencyIdStr)){
            response.setCode(2);
            response.setMessage("参数错误");
            return response;
        }

        int currencyId = Integer.parseInt(currencyIdStr);

        UserCoinConfigVO userCoinConfig = null;
        List<UserCoinConfigVO> userCoinConfigList = jydpCoinConfigService.listUserCoinConfigByUserId(userBo.getUserId());
        if (userCoinConfigList != null && userCoinConfigList.size() > 0) {
            for (UserCoinConfigVO userCoinConfigVO : userCoinConfigList) {
                if (userCoinConfigVO.getCurrencyId() == currencyId) {
                    userCoinConfig = userCoinConfigVO;
                }
            }
        }

        if (userCoinConfig == null) {
            response.setCode(3);
            response.setMessage("网络错误,请刷新页面重试");
            return  response;
        }

        JSONObject jsonObject = new JSONObject();
        jsonObject.put("userCoinConfig", userCoinConfig);

        response.setCode(1);
        response.setMessage("查询成功");
        response.setData(jsonObject);
        return  response;
    }

    /**  提币 */
    @RequestMapping(value = "mentionCoin.htm")
    public @ResponseBody
    JsonObjectBO mentionCoin(HttpServletRequest request) {
        JsonObjectBO response = new JsonObjectBO();

        UserSessionBO userBo = UserWebInterceptor.getUser(request);
        if (userBo == null) {
            response.setCode(4);
            response.setMessage("未登录");
            return response;
        }

        String currencyIdStr = StringUtil.stringNullHandle(request.getParameter("currencyId"));
        String numberStr = StringUtil.stringNullHandle(request.getParameter("number"));
        String validateCode = StringUtil.stringNullHandle(request.getParameter("validateCode"));
        String buyPwd = StringUtil.stringNullHandle(request.getParameter("buyPwd"));
        buyPwd = Base64Util.decode(buyPwd);

        if (!StringUtil.isNotNull(currencyIdStr) || !StringUtil.isNotNull(numberStr)
                || !StringUtil.isNotNull(validateCode) || !StringUtil.isNotNull(buyPwd)) {
            response.setCode(2);
            response.setMessage("参数错误");
            return response;
        }

        int currencyId = Integer.parseInt(currencyIdStr);
        double number = Double.parseDouble(numberStr);

        if (number <= 0) {
            response.setCode(3);
            response.setMessage("提现数量必须大于0");
            return response;
        }

        if (number % 1 > 0) {
            String suf = numberStr.substring(numberStr.indexOf('.') + 1, numberStr.length());
            if (suf.length() > 2) {
                response.setCode(3);
                response.setMessage("请输入正确的提币数量");
                return response;
            }
        }

        UserDO user = userService.getUserByUserId(userBo.getUserId());
        if (user == null) {
            response.setCode(3);
            response.setMessage("该用户不存在");
            return response;
        }
        if (user.getAccountStatus() != 1) {
            response.setCode(3);
            response.setMessage("该账号已被禁用");
            return response;
        }

        //查询用户是否有电子钱包帐户
//        SylUserBoundDO sylUserBound = sylUserBoundService.getSylUserBoundByUserId(userBo.getUserId());
//        if (sylUserBound == null) {
//            response.setCode(3);
//            response.setMessage("该账号未绑定电子钱包帐户");
//            return response;
//        }

        //查询用户币数量
        UserCurrencyNumDO userCurrencyNum = userCurrencyNumService.getUserCurrencyNumByUserIdAndCurrencyId(userBo.getUserId(), currencyId);
        if(userCurrencyNum == null){
            response.setCode(3);
            response.setMessage("币种信息获取失败,请稍候再试");
            return response;
        }
        if (userCurrencyNum.getCurrencyNumber() < number) {
            response.setCode(3);
            response.setMessage("提现数量大于用户币种数量");
            return response;
        }

        //验证码判定
        JsonObjectBO validatePhone = systemValidatePhoneService.validatePhone(user.getPhoneAreaCode() + user.getPhoneNumber(), validateCode);
        if(validatePhone.getCode() != 1){
            response.setCode(validatePhone.getCode());
            response.setMessage(validatePhone.getMessage());
            return response;
        }

        buyPwd = MD5Util.toMd5(buyPwd);
        boolean checkResult = userService.validateUserPay(user.getUserAccount(), buyPwd);
        if (!checkResult) {
            response.setCode(4);
            response.setMessage("交易密码错误");
            return response;
        }

        JydpCoinConfigDO jydpCoinConfig = jydpCoinConfigService.getJydpCoinConfigByCurrencyId(currencyId);
        if (jydpCoinConfig == null) {
            response.setCode(3);
            response.setMessage("币种管理信息错误");
            return response;
        }
        if (jydpCoinConfig.getMinCurrencyNumber() > number) {
            response.setCode(3);
            response.setMessage("币种提现数量不能小于币种最低提现数量");
            return response;
        }

        // TODO: 2018/3/30 0030  aaaaa --> sylUserBound.getUserSylAccount()
        boolean resultBoo = jydpUserCoinOutRecordService.insertJydpUserCoinOutRecord(currencyId, jydpCoinConfig.getCurrencyName(), userBo.getUserId(),
                userBo.getUserAccount(), "aaaaa", number);

        if (resultBoo) {
            response.setCode(1);
            response.setMessage("操作成功!");
        } else {
            response.setCode(2);
            response.setMessage("操作失败!");
        }

        return  response;
    }
}
