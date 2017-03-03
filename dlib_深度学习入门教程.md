#dlib_深度学习入门笔记
一份简要的笔记
##简单网络定义
dlib的网络采用一系列的模板进行组合而成。  
     
    using net_type = loss_multiclass_log< //多类别输出
                                fc<10,        
                                relu<fc<84,   //将84个全连接层的输出至relu激活器
                                relu<fc<120,  //将120个全连接层的输出至relu激活器
                                max_pool<2,2,2,2,relu<con<16,5,5,1,1,
                                max_pool<2,2,2,2,relu<con<6,5,5,1,1,
                                input<matrix<unsigned char>> 
                                >>>>>>>>>>>>; 

	loss_multiclass_log<SUBNET>  	//输出概率最大的类别
	fc<n, SUBNET>  					//全连接后生成n个类别
	relu<SUBNET>					//激活层
	max_pool<nr, nc, stride_y, stride_x, SUBNET> //rxc的极大池化，步长为r,c
	cnn<filter_number, nr, nc, r, c, SUBNET> //具有filter_number个rxc的卷积核，步长为r,c
	input<matrix<unsigned char>>			//输入类型，但是彩色图像不能这么输入
	input_rgb_image_sized<NR, NC=NR>				//rgb图像输入方式				
	
	matrix<unsigned char> //灰度图定义方式
	matrix<rgb_pixel> //彩色图定义方式

至此就可以利用上述构建完成一个简单的网络搭建了。

##网络训练
不同的网络定义，标签不同，如上文的网络，数据与标签为：
  
    std::vector<matrix<unsigned char>> training_images;  
    std::vector<unsigned long>         training_labels;  
	
构建好网络之后，就要进行网络训练参数的设置了。

    dnn_trainer<net_type> trainer(net); //利用网络初始化一个训练器
    trainer.set_learning_rate(0.01);	//设置学习速率
    trainer.set_min_learning_rate(0.00001);	//设置最低学习速率
    trainer.set_mini_batch_size(128);	//设置mini_batch_size
    trainer.be_verbose();				//设置冗余训练，在训练时会保存冗余结果，这样关闭可以继续训练
	trainer.set_synchronization_file("mnist_sync", std::chrono::seconds(20)); //设置冗余结果存储位置，以及储存周期
	trainer.train(training_images, training_labels);  //设置训练图像，已经表情
	net.clean();		//清楚无关数据，减少不必要的网络信息，有利于网络保存

##使用
	serialize(file_path) << net;		//存储网络
	deserialize(file_path) >> net;		//导入网络

使用网络进行分类
	
	net_type net;
	std::vector<unsigned long> predicted_labels = net(training_images);

使用网络获得概率
	
	anet_type net;
	//进行net的训练或者网络读入
    softmax<anet_type::subnet_type> snet; 
    snet.subnet() = net.subnet();
	dlib::array<matrix<rgb_pixel>> images;
	matrix<float,1,K> p = mat(snet(images.begin(), images.end())));

##更强大的网络定义
在定义更加强大的网络之前，先介绍几个接口。

add_prev

	//layers_abstract.h
	template <template<typename>class tag>
	class add_prev_{};//将前面两层的数据简单的加起来（前一层，和前面tag所指向的层）
	using add_prev = add_layer<add_prev_<tag>, SUBNET>;
	using add_prev1_  = add_prev_<tag1>;
	
tag

	//core_abstract.h
	template < unsigned long ID, typename SUBNET>
	class add_tag_layer{};//本层不作任何计算，但是具有层的名称，可以供其它层使用
	template <typename SUBNET> using tag1  = add_tag_layer< 1, SUBNET>;

skip
	
	//core_abstract.h
	template <template<typename> class TAG_TYPE, typename SUBNET>
    class add_skip_layer{};//从TAG_TYPE中获取输入，并完成身份转变？
	template <typename SUBNET> using skip1  = add_skip_layer< tag1, SUBNET>;

repeat

	//core_abstract.h
	template <size_t num, template<typename> class REPEATED_LAYER, typename SUBNET>
    class repeat {};将REPEATED_LAYER网络重复num次后，接SUBNET网络

