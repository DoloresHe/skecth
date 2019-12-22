### api开发文档

# /password/email
输入格式
email

输出格式
200
403
404
422

### 发送重置邮件
1. 注册url
/api.php
Route::post('password/email', 'Auth\ForgotPasswordController@sendResetLinkEmail');

2. 响应函数
Auth\ForgotPasswordController@sendResetLinkEmail
    public function sendResetLinkEmail(Request $request)
    {
//重写了validator,为了规范返回格式
       $validator = Validator::make($request->all(), [
        'email' => 'required|email|email'
    ]);
    if ($validator->fails()) {
        $data = [
            "message"=> "validation failed"
       ];
        abort(422,json_encode($data));
    }
        $user_check = User::where('email', $request->email)->first();
        $data = [
            'email'   => $request->email
        ];
//该邮箱账户不存在
        if (!$user_check) {        
            abort(404,($data));
        }
//当日注册的用户不能重置密码
        if ($user_check->created_at > Carbon::now()->subDay(1)){     
            abort(403,($data));
        }

        $email_check = DB::table('password_resets')->where('email', $request->email)->first();
 
//该邮箱12小时内已发送过重置邮件。请不要重复发送邮件，避免被识别为垃圾邮件。 这个应该可以放到redis里面
        if ($email_check&&$email_check->created_at>Carbon::now()->subHours(12)){
            abort(403);
        }

        $response = $this->broker()->sendResetLink(
            $request->only('email')
        );

        return $response == Password::RESET_LINK_SENT
        ? response()->success(($data))
        : response()->error(config('error.595'), 595);
    }

3. 使用自定义notification
/User.php
    public function sendPasswordResetNotification($token) 
    { 
     $this->notify(new ResetPasswordNotification($token)); 
    } 
    
4. 重写notification
主要是toMail函数
    public function toMail($notifiable)
    {
        return (new MailMessage)
        ->subject('废文网密码重置/重激活')
        ->line('以下是你的废文网密码重置邮件链接')
        ->action('密码重置/重激活', ("https://sosad.fun/password/reset/".$this->token),false)
        //->action('密码重置/重激活', url(route('password.reset', $this->token),false))
        ->line('如果你没有发出密码重置请求，请忽视此邮件');

    }
    
    
### 重置密码
输入参数
email
password
token

返回参数
200

1. 注册api
Route::post('password/reset', 'API\PassportController@postReset');

2. 响应函数
//reset 把新的密码写入数据库
    protected function reset(array $data)
    {
        $user = User::updateOrCreate(
            ['email'=>$data['email']],
            ['password' => bcrypt($data['password']), 'email_verified_at' => Carbon::now(),'remember_token' => Str::random(60)]);
        return $user;
    }
   //校验token是否有效，密码是否格式正确
    public function postReset(Request $request)
    {
        $password = $request->password;
        $data = $request->all();
        $rules = [
            'password'=>'required|between:6,20',
        ];
    
        $messages = [
            'required' => '密码不能为空',
            'between' => '密码必须是6~20位之间'
        ];
        $validator = Validator::make($data, $rules, $messages);

        if ($validator->fails()) {
            abort(403);
            //返回一次性错误
        }
        $token=hash::make($request->token);
        $token_check = DB::table('password_resets')->where('email',$request->email)->first();
        if(!hash::check($request->token,$token_check->token))
            abort(404,$token);
            //token不存在
        if ($token_check&&$token_check->created_at<Carbon::now()->subMinutes(30)){
            abort(403);
          //  token过期
        }
        $user=$this->reset($data);

   return response()->success('200');

    }  

