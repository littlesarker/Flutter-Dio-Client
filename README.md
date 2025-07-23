import 'dart:async';
import 'dart:io';
import 'package:connectivity_plus/connectivity_plus.dart';
import '../routes/app_pages.dart';
import '../utils/file.dart';
import '../utils/loader.dart';
import 'package:dio/dio.dart' as dio;
import 'package:dio/dio.dart';
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:image_picker/image_picker.dart';
import 'package:path_provider/path_provider.dart';
import 'package:permission_handler/permission_handler.dart';

import '../config/api_constant.dart';
import '../data/login_creadential.dart';
import '../model/api_response.dart';
import '../utils/network.dart';
import '../utils/snackbar.dart';

class ApiCommunication {
  late dio.Dio _dio;
  final String baseUrl = ApiConstant.BASE_URL;
  late Map<String, dynamic> header;
  late Connectivity connectivity;
  late LoginCredential _loginCredential;
  late String? token;

  ApiCommunication({int? connectTimeout, int? receiveTimeout}) {
    _dio = Dio();
    _dio.options.connectTimeout =
        Duration(milliseconds: connectTimeout ?? 60000); //1000 = 1s
    _dio.options.receiveTimeout =
        Duration(milliseconds: receiveTimeout ?? 60000);

    connectivity = Connectivity();
    _loginCredential = LoginCredential();
    token = _loginCredential.getToken();
    header = {
      'Accept': '*/*',
      'Authorization': 'Bearer $token',
    };
  }

  Future<bool> isConnectedToInternet() async {
    final List<ConnectivityResult> connectivityResult =
        await (Connectivity().checkConnectivity());

    if (connectivityResult.contains(ConnectivityResult.mobile) ||
        connectivityResult.contains(ConnectivityResult.wifi)) {
      return true;
    } else {
      return false;
    }
  }

  Future<ApiResponse> doGetRequest({
    required String apiEndPoint,
    Map<String, dynamic>? queryParameters,
    bool enableLoading = false,
    String? errorMessage,
  }) async {
    if (await isConnectedToInternet()) {
      if (enableLoading) showLoader();
      String requestUrl = '$baseUrl$apiEndPoint';
      debugPrint(
          '--------------------------------------Get Request------------------------------------------------');
      debugPrint('Request Url: $requestUrl $queryParameters');
      try {
        final response = await _dio.get(
          requestUrl,
          queryParameters: queryParameters,
          options: dio.Options(headers: header),
        );
        if (enableLoading) dismissLoader();
        if (response.statusCode == 200) {
          Map<String, dynamic> responseData = response.data;
          printSuccessResponse('$responseData');
          return ApiResponse(
            isSuccessful: true,
            statusCode: 200,
            message: responseData['message'],
            status: responseData['status'],
            data: responseData['data'],
          );
        } else {
          // Error Response
          showErrorSnackkbar(
            message: 'Exceptional status code found',
            titile: '${response.statusCode}',
          );
          printErrorResponse('${response.statusCode}');
          return ApiResponse(
            isSuccessful: false,
            status: 'unknown',
            message: 'Exceptional status code found',
          );
        }
      } on DioException catch (error) {
        if (enableLoading) dismissLoader();
        // Retriving Response Error
        if (error.response?.statusCode == 401) {
          printErrorResponse('Token is invalid');
          showErrorSnackkbar(
            titile: 'Token  Expired!',
            message: 'You need to login again',
          );
          // _loginCredential.clearLoginCredentialCache();
          // Get.offAllNamed(Routes.SPLASH);
          return ApiResponse(
            isSuccessful: false,
            statusCode: 401,
            status: 'error',
            message: 'Token is invalid',
          );
        } else {
          String errorStatus = 'unknown';
          String errorMessage = 'Server Exception';
          try {
            Map<String, dynamic> messageMap =
                error.response?.data as Map<String, dynamic>;
            errorStatus = messageMap['status'];
            errorMessage = messageMap['message'];
          } catch (error) {}

          printErrorResponse(errorMessage);
          showErrorSnackkbar(message: errorMessage);

          return ApiResponse(
            isSuccessful: false,
            status: errorStatus,
            message: errorMessage,
          );
        }
      } on SocketException catch (error) {
        if (enableLoading) dismissLoader();

        printErrorResponse(error.message);
        showErrorSnackkbar(
          titile: 'Socket Exception',
          message: error.message,
        );

        return ApiResponse(
          isSuccessful: false,
          status: 'error',
          message: error.message,
        );
      } catch (error) {
        if (enableLoading) dismissLoader();
        showErrorSnackkbar(message: 'Unkown Exception');
        printErrorResponse(error.toString());
        return ApiResponse(
          isSuccessful: false,
          status: 'unkown',
          message: 'Unkown Exception',
        );
      }
    } else {
      errorMessage = 'You are not connected with mobile/wifi network';
      showWarningSnackkbar(message: errorMessage);
      debugPrint(errorMessage);
      //! Returning Network Not found Error Response
      return ApiResponse(
        isSuccessful: false,
        statusCode: 503,
        message: errorMessage,
      );
    }
  }

  Future<ApiResponse> doPostRequest({
    required String apiEndPoint,
    Map<String, dynamic>? requestData,
    Map<String, dynamic>? queryParameters,
    List<XFile>? xFiles,
    List<String>? xFilesKeyes,
    List<File>? files,
    List<String>? fileKeyes,
    String? errorMessage,
    bool enableLoading = false,
    bool isFormData = false,
  }) async {
    dio.Response? response;
    String requestUrl = '$baseUrl$apiEndPoint';
    debugPrint(
        '--------------------------------------Post Request------------------------------------------------');
    debugPrint(
        'Request Url: $requestUrl\n\nIs Form Request: $isFormData \n\n Request Data: $requestData\n');

    if (await isConnectedToInternet()) {
      //? Internet Connection is available
      if (enableLoading) showLoader();
      final Object? requestObject;
      //==================================================== Checking if request data is for Form or not
      if (isFormData) {
        //* If request data is Form data
        dio.FormData requestFormData = dio.FormData.fromMap(requestData ?? <String, dynamic>{});
        //* Check for Xfile
        if (xFiles != null && xFiles.isNotEmpty) {
          // Having XFile to attatch
          List multiparts = await getMultipartFilesFromXfiles(xFiles);
          for (int i = 0; i < multiparts.length; i++) {
            requestFormData.files
                .add(MapEntry(xFilesKeyes?[i] ?? 'image', multiparts[i]));
          }
        }
        //* Check for File
        if (files != null && files.isNotEmpty) {
          // Having File to attach
          for (int i = 0; i < files.length; i++) {
            requestFormData.files.add(
              MapEntry(
                fileKeyes?[i] ?? 'images[0][$i]',
                dio.MultipartFile.fromFileSync(
                  files[i].path,
                  contentType: getMediaTypeFromFilePath(files[i].path),
                  filename: getFileNameFromFile(files[i]),
                ),
              ),
            );
          }
          // generateCurlCommandFromDioFormRequest(
          //   url: requestUrl,
          //   headers: header,
          //   formData: requestData,
          //   files: requestFormData.files,
          // );
        }

        //* Assgining from data in request body
        requestObject = requestFormData;
      } else {
        //* If request data is Raw data
        requestObject = requestData;
      }
      // Request data updated as parameters =================================================================

      try {
        response = await _dio.post(
          requestUrl,
          data: requestObject,
          queryParameters: queryParameters,
          options: dio.Options(headers: header),
        );
        //* ============================================================== Handling Success Response
        if (enableLoading) dismissLoader();

        if (response.statusCode == 200) {
          //? Success Response
          Map<String, dynamic> responseData = response.data;
          printSuccessResponse('$responseData');
          return ApiResponse(
            isSuccessful: true,
            statusCode: 200,
            message: responseData['message'],
            status: responseData['status'],
            data: responseData['data'],
          );
        } else {
          // Error Response
          showErrorSnackkbar(
            message: 'Exceptional status code found',
            titile: '${response.statusCode}',
          );
          printErrorResponse('${response.statusCode}');
          return ApiResponse(
            isSuccessful: false,
            status: 'unknown',
            message: 'Exceptional status code found',
          );
        }
        //* Success Response Handled=================================================================
      }

      //!================================================================ Handing Error Response Handled
      on DioException catch (error) {
        if (enableLoading) dismissLoader();
        // Retriving Response Error
        if (error.response?.statusCode == 401) {
          printErrorResponse('Token is invalid');
          showErrorSnackkbar(
            titile: 'Token  Expired!',
            message: 'You need to login again',
          );
          _loginCredential.clearLoginCredentialCache();
          Get.offAllNamed(Routes.SPLASH);
          return ApiResponse(
            isSuccessful: false,
            statusCode: 401,
            status: 'error',
            message: 'Token is invalid',
          );
        } else {
          String errorStatus = 'unknown';
          String errorMessage = 'Server Exception';
          try {
            Map<String, dynamic> messageMap =
                error.response?.data as Map<String, dynamic>;
            errorStatus = messageMap['status'];
            errorMessage = messageMap['message'];
          } catch (error) {}

          printErrorResponse(errorMessage);
          showErrorSnackkbar(message: errorMessage);

          return ApiResponse(
            isSuccessful: false,
            status: errorStatus,
            message: errorMessage,
          );
        }
      } on SocketException catch (error) {
        if (enableLoading) dismissLoader();

        printErrorResponse(error.message);
        showErrorSnackkbar(
          titile: 'Socket Exception',
          message: error.message,
        );

        return ApiResponse(
          isSuccessful: false,
          status: 'error',
          message: error.message,
        );
      } catch (error) {
        if (enableLoading) dismissLoader();
        showErrorSnackkbar(message: 'Unkown Exception');
        printErrorResponse(error.toString());
        return ApiResponse(
          isSuccessful: false,
          status: 'unkown',
          message: 'Unkown Exception',
        );
      }
      //! Error Response Handled=================================================================
    } else {
      //! Internet Not Vailable
      showWarningSnackkbar(
        titile: 'Connection Error!',
        message: 'You are not connected with mobile/wifi network',
      );
      printErrorResponse('You are not connected with mobile/wifi network');
      return ApiResponse(
        isSuccessful: false,
        statusCode: 503,
        status: '',
        message: 'You are not connected with mobile/wifi network',
      );
    }
  }

  void printSuccessResponse(String text) {
    debugPrint('Success Response: $text');
  }

  void printErrorResponse(String text) {
    debugPrint('Error Response: $text');
  }

  void endConnection() => _dio.close(force: true);

  Stream<double> downloadFile(String url) {
    StreamController<double> progressController = StreamController<double>();
    doDownloadingTask(url, progressController);
    return progressController.stream;
  }

  Future<void> doDownloadingTask(
      String url, StreamController<double> progressController) async {
    final filePath = await getDownloadsDirectoryPath(url);
    try {
      var dio = Dio();
      await dio.download(url, filePath, onReceiveProgress: (received, total) {
        if (total != -1) {
          double progress = (received / total * 100);
          progressController.add(progress);
        }
      });

      print('Download completed and saved to: $filePath');
    } catch (e) {
      print('Error downloading image: $e');
    }
  }

  Future<String> getDownloadsDirectoryPath(String url) async {
    var status = await Permission.storage.status;
    if (!status.isGranted) {
      await Permission.storage.request();
    }
    if (Platform.isAndroid) {
      return "/storage/emulated/0/Download/${url.split('/').last}";
    }
    if (Platform.isIOS) {
      final directory = await getApplicationDocumentsDirectory();
      return '${directory.path}${url.split('/').last}';
    }
    throw UnsupportedError("Platform not supported");
  }
}
