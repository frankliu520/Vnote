# XC7K325t和RK3568pcie通信

1.1. 概述
本文档，描述了rk3568和FPGA通过pcie2.0通信的研发使用过程，完成fpga程序动态升级的功能，升级方式采用Program_B复位 AS+spi的方式。

1.2. 准备工作
1.2.1. 硬件平台
本次硬件平台为瑞芯微RK3568+xilinx xc7k325T 产品设计之初始考虑了全国产的使用场景。

1.2.2. 软件平台
搭建飞凌开发环境，可以自己搭建也可以用官方提供好的虚拟机，注意使用虚拟机或服务器，电脑配置大于内存大于16G,硬盘容量大于500G
下载飞凌嵌入式内核源码，先进行全编译，需要编译文件系统时，需单独编译，编译完成后会将生成的rootfs.img替换linuxfs/rootfs.img文件，后续执行build.sh仍然使用linuxfs/rootfs.img文件。具体参考官方指导文档链接此处目的是为了获取交叉编译工具，为后续qt/svcode交差编译环境搭建提供基础，编译buildroot时间比较长耐心等待

1.3. 修改设备树
设备树保持原状其实就可以 注意spi的设备树配置一定要开启注意最大频率其配置参考

📋
&spi0 {
	pinctrl-names = "default", "high_speed";
	pinctrl-0 = <&spi0m1_cs0 &spi0m1_pins>;
	pinctrl-1 = <&spi0m1_cs0 &spi0m1_pins_hs>;
	status = "okay";

	spi@0 {
		compatible = "rockchip,spidev";
		reg = <0>;
		spi-max-frequency = <50000000>;
	};
};

&spi2 {
	pinctrl-names = "default", "high_speed";
	pinctrl-0 = <&spi2m1_cs0 &spi2m1_cs1 &spi2m1_pins>;
	pinctrl-1 = <&spi2m1_cs0 &spi2m1_cs1 &spi2m1_pins_hs>;
	status = "okay";

	spi@0 {
		compatible = "rockchip,spidev";
		reg = <0>;
		spi-max-frequency = <50000000>;
	};

	spi@1 {
		compatible = "rockchip,spidev";
		reg = <1>;
		spi-max-frequency = <50000000>;
	};
};
1.4. 升级代码
1.4.1. h文件
1.4.1.1. fpga.h文件
📋
/*******************************************************************************
 * FileName:fpga.h
 * Author: dll    Date:2025-10-23
 * Description:
 *	some code copy from l.
 * History:
 *     <author>        <time>        <desc>
 *
 ******************************************************************************/
#ifndef  __MAND_MANAGE_FPGA_H
#define  __MAND_MANAGE_FPGA_H


#define  FPGA_DEVICE_NAME  "/dev/spidev2.0"
#include <stdint.h>
/* 引脚号（RK3568 实测） */
#define FPGA_PIN_NCONFIG_GPIO   98   // GPIO3_A2  PROGRAM_B
#define FPGA_PIN_NSTATUS_GPIO   99   // GPIO3_A3  INIT_B
#define FPGA_PIN_CONF_DOWN_GPIO 97   // GPIO3_A1  DONE

#define BLOCK_SZ 1024

int  open_spi_device(int trycnt);
int open_spi_drv_file( void );
void close_spi_drv_file( void );
int config_spi_dn_fpga( void );
int write_spi_dn_fpga_data( unsigned char *txbuf,unsigned char *rxbuf, int buf_len );
int free_spi_dn_fpga( void );
int get_fpga_conf_down(int *over);
int get_fpga_nstatus(int *status);

///TOPAPI 使用结构体 和 函数声明  //////////////////////////////
struct manageFpgaLoadReq {
	char filePath[100];     /**< FPGA 加载文件路径, 必须为字符串  */
};

struct manageFpgaUpdateReq {
	char tmpFilePath[100];  /**< 用临时的FPGA文件替换FPGA code, 必须为字符串 */
};



#ifdef __cplusplus
extern "C" {
#endif

// 这里只声明C++需要调用的C函数
int fpga_node_init(); //c++调用外部接口
int test_standard_spi(void) ;

// 如果C++还需要调用其他C函数，可以在这里添加

#ifdef __cplusplus
}
#endif

int fpga_node_free(void);
#endif

1.4.1.2. include "lr_errors.h文件
📋
#ifndef LR_ERRORS_H_
#define LR_ERRORS_H_

/* 常用错误码，可按需要继续扩充 */
#define LR_OK                0
#define LR_FAILD            -1
#define LR_ERR ioctl        -2

#endif /* LR_ERRORS_H_ */
1.4.2. .c文件
1.4.2.1. fpga.c
📋

/*******************************************************************************
 * FileName:fpga.c
 * Author: dll    Date:2025-10-23
 * Description:
 *	some code copy from l.
 * History:
 *     <author>        <time>        <desc>
 *
 ******************************************************************************/
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/time.h>
#include <string.h>
#include <errno.h>
#include <sys/ioctl.h>
#include <pthread.h>
#include <sys/prctl.h>
#include "lr_errors.h"
#include <stdlib.h>
#include <linux/spi/spidev.h>
#include "fpga.h"

#define DEFAULT_FPGA_FILE  "/root/fpga.bit"
int g_spi_drv_dbg =0;
#include "lr_errors.h"
//#include "mand.h"

static int gl_spi_fd = -1;

static int gpio_export(int pin)
{
    char buffer[64];
    int fd, len;
    
    fd = open("/sys/class/gpio/export", O_WRONLY);
    if (fd < 0) {
        printf("Failed to open GPIO export: %s\n", strerror(errno));
        return -1;
    }
    
    len = snprintf(buffer, sizeof(buffer), "%d", pin);
    write(fd, buffer, len);
    close(fd);
    
    usleep(100000); // 等待100ms让系统创建GPIO文件
    return 0;
}

static int gpio_unexport(int pin)
{
    char buffer[64];
    int fd, len;
    
    fd = open("/sys/class/gpio/unexport", O_WRONLY);
    if (fd < 0) {
        printf("Failed to open GPIO unexport: %s\n", strerror(errno));
        return -1;
    }
    
    len = snprintf(buffer, sizeof(buffer), "%d", pin);
    write(fd, buffer, len);
    close(fd);
    
    return 0;
}

static int gpio_set_direction(int pin, const char *direction)
{
    char path[64];
    int fd;
    
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%d/direction", pin);
    
    // 重试几次，因为GPIO导出后文件可能不会立即创建
    int retries = 5;
    while (retries--) {
        fd = open(path, O_WRONLY);
        if (fd >= 0) break;
        usleep(100000); // 等待100ms
    }
    
    if (fd < 0) {
        printf("Failed to set GPIO%d direction: %s\n", pin, strerror(errno));
        return -1;
    }
    
    write(fd, direction, strlen(direction));
    close(fd);
    
    return 0;
}

static int gpio_set_value(int pin, int value)
{
    char path[64];
    int fd;
    
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%d/value", pin);
    fd = open(path, O_WRONLY);
    if (fd < 0) {
        printf("Failed to set GPIO%d value: %s\n", pin, strerror(errno));
        return -1;
    }
    
    write(fd, value ? "1" : "0", 1);
    close(fd);
    
    return 0;
}

static int gpio_get_value(int pin, int *value)
{
    char path[64];
    char ch;
    int fd;
    
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%d/value", pin);
    fd = open(path, O_RDONLY);
    if (fd < 0) {
        printf("Failed to get GPIO%d value: %s\n", pin, strerror(errno));
        return -1;
    }
    
    read(fd, &ch, 1);
    close(fd);
    
    *value = (ch == '1') ? 1 : 0;
    return 0;
}



/**
 * #define FPGA_PIN_NCONFIG_GPIO   98   // GPIO3_A2  PROGRAM_B
   #define FPGA_PIN_NSTATUS_GPIO   99   // GPIO3_A3  INIT_B
  #define FPGA_PIN_CONF_DOWN_GPIO 97   // GPIO3_A1  DONE
 */
static int config_gpio_for_dn_fpga( void )
{

    int value = -1;
    
    printf("Configuring GPIO for FPGA download...\n");
    
    // 导出GPIO引脚
    printf("Exporting GPIO pins...\n");
    gpio_export(FPGA_PIN_CONF_DOWN_GPIO);
    gpio_export(FPGA_PIN_NSTATUS_GPIO);
    gpio_export(FPGA_PIN_NCONFIG_GPIO);
    
    // 设置方向
    printf("Setting GPIO directions...\n");
    gpio_set_direction(FPGA_PIN_CONF_DOWN_GPIO, "in");
    gpio_set_direction(FPGA_PIN_NSTATUS_GPIO, "in");
    gpio_set_direction(FPGA_PIN_NCONFIG_GPIO, "out");
    usleep(1000);
    // 读取初始状态
    gpio_get_value(FPGA_PIN_NSTATUS_GPIO, &value);
    printf("INIT_B-------0 %d\n", value);
    
    gpio_get_value(FPGA_PIN_CONF_DOWN_GPIO, &value);
    printf("DONE--------------: %d\n", value);
    
    // FPGA配置序列
    printf("Starting FPGA configuration sequence...\n");
    gpio_set_value(FPGA_PIN_NCONFIG_GPIO, 0);  // 拉低PROGRAM_B

	usleep(4000);

    // 检查配置后的状态
    gpio_get_value(FPGA_PIN_NSTATUS_GPIO, &value);
    printf( "INIT_B:--------1 %d\n", value); //INIT_B
	usleep(1000);
	gpio_set_value(FPGA_PIN_NCONFIG_GPIO, 1);  // 拉高PROGRAM_B
	usleep(4000); 
	    // 检查配置后的状态
    gpio_get_value(FPGA_PIN_NSTATUS_GPIO, &value);
    printf( "INIT_B:--------2 %d\n", value); //INIT_B
	usleep(1000);
	return 0 ;
}



int  open_spi_device(int trycnt)
{
	int fd;
	int docnt = trycnt;
	while(docnt--){
		fd = open( FPGA_DEVICE_NAME, O_RDWR ) ;
		if( fd < 0 ) {
			//ERROR(DEBUG_MANAGE,"open device " FPGA_DEVICE_NAME " error\n" ) ;
			usleep(50);
		}else{
			return fd;
		}
	}
	return -1;
}


int open_spi_drv_file( void )
{
	gl_spi_fd = open_spi_device(20000);

	if( gl_spi_fd < 0 ) 
		return LR_FAILD;
	else
		return LR_OK;
}

void close_spi_drv_file( void )
{
	if( gl_spi_fd > 0 ) 
	{
		close( gl_spi_fd );
		gl_spi_fd = -1;
	}
	return;
}


int config_spi_dn_fpga( void )
{
    int ret = LR_OK;
    uint8_t mode = 0;        // SPI模式0
    uint8_t bits = 8;        // 8位数据
    uint32_t speed = 2000000; // 10MHz
    
    if(gl_spi_fd < 0) 
        return LR_FAILD;

    printf("Configuring SPI for FPGA download...\n");


    // 设置SPI模式
    ret = ioctl(gl_spi_fd, SPI_IOC_WR_MODE, &mode);
    if(ret < 0) {
        printf("Set SPI mode failed: %s\n", strerror(errno));
        return LR_FAILD;
    }

    // 设置时钟频率
    ret = ioctl(gl_spi_fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);
    if(ret < 0) {
        printf("Set SPI speed failed: %s\n", strerror(errno));
        return LR_FAILD;
    }

    // 设置每字位数
    ret = ioctl(gl_spi_fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    if(ret < 0) {
        printf("Set SPI bits per word failed: %s\n", strerror(errno));
        return LR_FAILD;
    }

    printf("SPI configured: mode=0, speed=10MHz, bits=8\n");
    return LR_OK;
}

int free_spi_dn_fpga( void )
{
    // 设置回较低的时钟频率
    uint32_t speed = 2000000;
    ioctl(gl_spi_fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);
    
    // 取消导出GPIO引脚
    gpio_unexport(FPGA_PIN_CONF_DOWN_GPIO);
    gpio_unexport(FPGA_PIN_NSTATUS_GPIO);
    gpio_unexport(FPGA_PIN_NCONFIG_GPIO);
    
    return LR_OK;

}


unsigned char turnbit(unsigned char val)
{
    unsigned char tmp = 0;
    unsigned char inval = val;
    int i = 0;
    for(i = 0; i < 7; i++){
        tmp |= inval & 0x01;
        tmp = tmp << 1;
        inval = inval >> 1;
    }
    tmp |= inval & 0x01;
    return tmp;
}


int write_spi_dn_fpga_data( unsigned char *txbuf, unsigned char  *rxbuf,int buf_len )
{
    int  ret;
    unsigned char wr_buf[1024];
    
    if(buf_len > 1024) {
        printf("Buffer too large: %d bytes\n", buf_len);
        return LR_FAILD;
    }

    // // 应用位反转
    // for(i = 0; i < buf_len; i++) {
    //     wr_buf[i] = turnbit(buf[i]);
    // }
    
    // 使用标准SPI传输
    struct spi_ioc_transfer xfer = {
        .tx_buf = (unsigned long)txbuf,
        .rx_buf = (unsigned long)rxbuf,   // 我们只发送，不接收
        .len = buf_len,
        .speed_hz = 10000000, // 10MHz
        .bits_per_word = 8,
        .delay_usecs = 50,
    };

    ret = ioctl(gl_spi_fd, SPI_IOC_MESSAGE(1), &xfer);

    if(ret < 0) { 
        printf("ERROR: SPI write failed: %s\n", strerror(errno));
        return LR_FAILD;
    }

    return LR_OK;
}

// ==================== FPGA状态读取函数 ====================

int get_fpga_conf_down(int *over)
{
    int value = -1;
    gpio_get_value(FPGA_PIN_CONF_DOWN_GPIO, &value);
    *over = value;
    printf("confdown:%d==============\n", value);
    return 0;
}

int get_fpga_nstatus(int *status)
{
    int value = -1;
    gpio_get_value(FPGA_PIN_NSTATUS_GPIO, &value);
    *status = value;
    printf("nstatus:%d======================\n", value);
    return 0;
}

// ==================== 文件操作函数 ====================

static int intf_get_file_length(const char *file, int *len)
{
    int ret;
    struct stat statbuf;
    
    ret = stat(file, &statbuf);
    if(ret == -1) {
        printf("Failed to get file stat: %s\n", strerror(errno));
        return LR_FAILD;
    }

    *len = statbuf.st_size;
    return LR_OK;
}




unsigned char rxbuf[1026];
// ==================== FPGA加载主函数 ====================

static int _do_fpga_load(void *param)
{
	int cnt = 0 ;
	unsigned char data_buf[BLOCK_SZ + 2];
    printf("do fpga load=============================\n");
    const char *file = (const char *)param;
    int fd_file;
    int ret, file_len;
    int over = -1;

    // 检查文件是否存在
    if(access(file, F_OK) != 0) {
        printf("FPGA file does not exist: %s\n", file);
        return LR_FAILD;
    }
    printf("access=============================\n");

    // 获取文件长度
    ret = intf_get_file_length(file, &file_len);
    if(ret != LR_OK) {
        printf("Failed to get file length\n");
        return LR_FAILD;
    }
    printf("File length: %d bytes\n", file_len);
    config_gpio_for_dn_fpga();
    // 打开SPI设备
    ret = open_spi_drv_file();
    if(ret < 0) {
        printf("Failed to open SPI device\n");
        return LR_FAILD;
    }

    // 配置SPI
    ret = config_spi_dn_fpga();
    if(ret < 0) {
        printf("Failed to configure SPI\n");
        close_spi_drv_file();
        return LR_FAILD;
    }

    // 打开FPGA比特流文件
    printf("Opening FPGA file...\n");
    fd_file = open(file, O_RDONLY);
    if(fd_file == -1) {
        printf("Failed to open FPGA file: %s\n", strerror(errno));
        close_spi_drv_file();
        return LR_FAILD;
    }

     memset(rxbuf,0xff,BLOCK_SZ);

    while( file_len > BLOCK_SZ )
    {
        ret = read( fd_file, (char *)data_buf, BLOCK_SZ);
        if( ret != BLOCK_SZ )
        {
            printf("m_file_read func ret error" ) ;
            goto over_process;

        }

        ret = write_spi_dn_fpga_data( data_buf, rxbuf,BLOCK_SZ) ;
        if( ret < 0 )
        {
			printf( "write_spi_dn_fpga_data faild" ) ;
            goto over_process;
        }
        file_len = file_len - BLOCK_SZ ;
        cnt++ ;
        if( ( cnt % 256 ) == 0 ){
            printf( ".") ;
            fflush(stdout);
        }

		if (g_spi_drv_dbg)
		{
			printf("SPI WRITE:\n  TX:  ");
			int i = 0;
			for (i = 0; i < BLOCK_SZ; i++)
			{
				printf("%02x ", rxbuf[i]);
			}
				printf("\n  RX:  ");
			for (i = 0; i < BLOCK_SZ; i++)
			{
				printf("%02x ", rxbuf[i]);
			}
				printf("\n");
		}
    }

    if( file_len > 0 )
    {
        ret = read( fd_file, (char *)data_buf, BLOCK_SZ);
        if( ret != file_len )
        {
            printf("m_file_read func ret error" ) ;
            goto over_process;
        }

        ret = write_spi_dn_fpga_data( data_buf, rxbuf,ret ) ;
        if( ret < 0 )
        {
			printf( "write_spi_dn_fpga_data faild" ) ;
            goto over_process;
        }
    }

    printf("Padding data sent\n");

    // 检查配置是否完成
    get_fpga_conf_down(&over);
    if(over != 1) {
        printf("FPGA configuration failed - DONE pin is not high\n");
        ret = LR_FAILD;
    } else {
        printf("FPGA configuration successful!\n");
        ret = LR_OK;
    }

    free_spi_dn_fpga();

over_process:
    printf("over_process:====================================\n");
    printf("FPGA load process ended with ret=%d\n", ret);
    printf("Closing SPI device and file...\n");
    
    // 释放资源
    close_spi_drv_file();
    if(fd_file != -1) close(fd_file);
    
    printf("Cleanup completed\n");
    return ret;
}

// static void do_fpga_load(void * param)
// {
// 	int ret = 0;
// 	/*TODO*/
// 	prctl(PR_SET_NAME, "download_fpga");
// 	ret = _do_fpga_load(param);
// 	pthread_exit(&ret);
// }

// static int _load_fpga(char *file)
// {
// 	pthread_t thread_fpga;

// 	//	DEBUG(FUNCTION_CALL_GROUP,"");
// 	pthread_create(&thread_fpga, NULL, (void *)do_fpga_load, (void *)file);

// 	return LR_OK;
// }

// ==================== 主接口函数 ====================

static char fpgaFile[100] = {0};

int fpga_node_init()
{
    const char *filePath = "/root/fpga.bit";
    
    if(filePath == NULL)
        strcpy(fpgaFile, DEFAULT_FPGA_FILE);
    else
        strcpy(fpgaFile, filePath);

    printf("Starting FPGA configuration from: %s\n", fpgaFile);
    return _do_fpga_load((void *)fpgaFile);
}

int fpga_node_free(void)
{
    return LR_OK;
}




int test_standard_spi(void) 
{
    int fd = open("/dev/spidev2.0", O_RDWR);
    if(fd < 0) {
        perror("Open standard SPI device");
        return -1;
    }
    
    // 设置标准SPI参数
    uint8_t mode = SPI_MODE_0;
    uint8_t bits = 8;
    uint32_t speed = 1000000;
    
    ioctl(fd, SPI_IOC_WR_MODE, &mode);
    ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);
    
    // 测试简单传输
    unsigned char tx[] = {0xAA, 0x55, 0x00};
    unsigned char rx[3];
    
    struct spi_ioc_transfer tr = {
        .tx_buf = (unsigned long)tx,
        .rx_buf = (unsigned long)rx,
        .len = 3,
    };
    
    int ret = ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
    printf("Standard SPI test: %s\n", ret < 0 ? "FAILED" : "PASSED");
    
    close(fd);
    return ret;
}
1.4.2.2. 测试加载程序
📋
    printf("FPGA Loader Test Program\n");
    
    // 先测试标准SPI
    printf("Testing standard SPI interface...\n");
    if(test_standard_spi() == 0) {
        printf("Standard SPI test passed\n");
    } else {
        printf("Standard SPI test failed\n");
        return -1;
    }
    
    // 运行FPGA加载
    printf("\nStarting FPGA loading process...\n");
    int result = fpga_node_init();
    

    return result;
1.5. 注意事项
序号	内容
1	注意根据实际使用情况分清使用的是标准spi还是使用的自定义spi,应该注意spi的延时时间设置
2	应该严格遵守Program_B复位 AS+spi 升级模式下的时序逻辑控制，当对应引脚状态正常后，再使能spi